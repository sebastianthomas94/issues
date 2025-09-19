# Cheque Payment GET Implementation

## Overview
This document outlines the implementation for displaying cheque payment status in the existing `GET /api/v1/x/students/:studentId/batches/:batchId/fees` endpoint.

## Changes Required

### 1. Update PaymentStatus Enum

**File**: `src/modules/purchased-batch-fee/core/enums/fee-payment-status.enum.ts`

```typescript
export enum PaymentStatus {
    Unpaid = "Unpaid",
    Incomplete = "Incomplete", // partially paid
    Paid = "Paid", // paid if cheque is cleared
    Refunded = "Refunded",
    Cancelled = "Cancelled",
    Unclear_R = "Unclear(R)", // cheque has been received from the user
    Unclear_P = "Unclear(P)", // cheque has been presented to the bank
    Voided = "Voided",
}
```

### 2. Create Cheque Status Mapping Utility

**File**: `src/modules/user/admin/student-batch/utils/cheque-status-mapper.util.ts`

```typescript
import { PaymentStatus } from "@modules/purchased-batch-fee/core/enums/fee-payment-status.enum";

export interface ChequeStatusInfo {
    chequeStatus: string;
    chequeNumber?: string;
    chequeDate?: string;
    presentationDate?: string;
    clearanceDate?: string;
    lastUpdatedBy?: string;
    lastStatusUpdate?: string;
    presentedReason?: string;
    receivedDocuments?: any[];
    presentedDocuments?: any[];
}

export function mapChequeStatusToPaymentStatus(
    orderNotes: Record<string, string>
): PaymentStatus {
    const chequeStatus = orderNotes.chequeStatus;
    const manualPaymentMethod = orderNotes.manualPaymentMethod;
    
    // Only apply cheque status mapping for cheque payments
    if (manualPaymentMethod !== "Cheque") {
        return PaymentStatus.Unpaid; // Default for non-cheque payments
    }
    
    switch (chequeStatus) {
        case "Received":
            return PaymentStatus.Unclear_R;
        case "Presented":
            return PaymentStatus.Unclear_P;
        case "Cleared":
            return PaymentStatus.Paid;
        case "Rejected":
        case "Bounced":
        case "Returned":
            return PaymentStatus.Unpaid;
        default:
            return PaymentStatus.Unpaid;
    }
}

export function extractChequeInfo(orderNotes: Record<string, string>): ChequeStatusInfo | null {
    if (orderNotes.manualPaymentMethod !== "Cheque") {
        return null;
    }
    
    return {
        chequeStatus: orderNotes.chequeStatus || "",
        chequeNumber: orderNotes.chequeNumber,
        chequeDate: orderNotes.chequeDate,
        presentationDate: orderNotes.presentationDate,
        clearanceDate: orderNotes.clearanceDate,
        lastUpdatedBy: orderNotes.lastUpdatedBy,
        lastStatusUpdate: orderNotes.lastStatusUpdate,
        presentedReason: orderNotes.presentedReason,
        receivedDocuments: orderNotes.receivedDocuments ? JSON.parse(orderNotes.receivedDocuments) : [],
        presentedDocuments: orderNotes.presentedDocuments ? JSON.parse(orderNotes.presentedDocuments) : [],
    };
}
```

### 3. Update Fee Component Construction Helper

**File**: `src/modules/purchased-batch-fee/core/helpers/construct-fee-components.helper.ts`

Add method to handle cheque payment status:

```typescript
import { mapChequeStatusToPaymentStatus } from "@modules/user/admin/student-batch/utils/cheque-status-mapper.util";

export function getPartialPaymentStatusWithCheque(
    item: IPurchaseHistory | IPurchaseHistoryInstalment,
    orderNotes?: Record<string, string>
): PaymentStatus {
    // First check if this is a cheque payment with special status
    if (orderNotes && orderNotes.manualPaymentMethod === "Cheque") {
        const chequeStatus = mapChequeStatusToPaymentStatus(orderNotes);
        
        // For cheque payments, return the cheque-specific status
        if ([PaymentStatus.Unclear_R, PaymentStatus.Unclear_P].includes(chequeStatus)) {
            return chequeStatus;
        }
        
        // If cleared, continue with normal payment status logic
        // If rejected/bounced/returned, treat as unpaid
        if (chequeStatus === PaymentStatus.Unpaid) {
            return PaymentStatus.Unpaid;
        }
    }
    
    // Default payment status logic (existing implementation)
    if (!item.paidDetails) {
        return PaymentStatus.Unpaid;
    }

    const totalPaid = item.paidDetails.reduce((sum, part) => sum + part.paidTotal, 0);
    const effectiveTotalAmount = item.kind === "instalment" ? item.amountTotal : getEffectiveTotalAmount(item.amountTotal, item.offers);

    if (totalPaid === 0) {
        return PaymentStatus.Unpaid;
    } else if (totalPaid < effectiveTotalAmount) {
        return PaymentStatus.Incomplete;
    } else {
        return PaymentStatus.Paid;
    }
}
```

### 4. Update Fee Structure Composition Utility

**File**: `src/modules/purchased-batch-fee/core/utils/construct-fee-components.util.ts`

Modify the `constructFeeComponentsForFeeStructure` function to include order notes for cheque status:

```typescript
export function constructFeeComponentsForFeeStructure(
    purchaseHistory: PurchaseHistoryNew[],
    feeStructureType: FeeStructureType,
    meta: { batchTitle: string; orderNotesMap?: Map<string, Record<string, string>> },
): (FeeComponent | PartialFeeComponent)[] {
    const filteredHistory = getFilteredPurchaseHistory(purchaseHistory, feeStructureType);
    const components: (FeeComponent | PartialFeeComponent)[] = [];

    filteredHistory.forEach((item, index) => {
        const isInstalment = item.kind === "instalment";
        const orderNotes = meta.orderNotesMap?.get(item.id);
        const status = getPartialPaymentStatusWithCheque(item, orderNotes);
        
        // Rest of the existing implementation...
        // Update status usage throughout the function
    });

    return components;
}
```

### 5. Extend Student Batch Service

**File**: `src/modules/user/admin/student-batch/services/student-batch.service.ts`

Update the `getFeeStructure` method to fetch and pass order notes:

```typescript
async getFeeStructure(studentId: string, batchId: string): Promise<GetFeeStructureResponseDto> {
    const purchasedBatch = await this.purchasedBatchService.findOneByBatchIdAndStudentId(batchId, studentId);

    if (!purchasedBatch) {
        throw new NotFoundException(`No purchased batch found for student ${studentId} and batch ${batchId}`);
    }

    if (purchasedBatch.type === PurchasedBatchType.Purchased || purchasedBatch.type === PurchasedBatchType.Added) {
        if (!purchasedBatch.batchAllocationId)
            throw new InternalServerErrorException("Batch allocation ID is missing for purchased batch");

        const batch = await this.readBatchService.findById(batchId);

        try {
            const historyResponse = await firstValueFrom(
                this.batchAllocationService.getPurchaseHistoryNew(
                    { id: purchasedBatch.batchAllocationId },
                    this.metadata,
                ),
            );

            // Fetch order notes for cheque status information
            const orderNotesMap = await this.fetchOrderNotesForPurchaseHistory(historyResponse.purchaseHistory);

            const feeStructure = composeSDFeeStructure(
                historyResponse.purchaseHistory, 
                batch.title,
                { orderNotesMap }
            );

            return {
                feeStructures: feeStructure,
                paymentDetails: generatePaymentDetails(historyResponse.purchaseHistory),
                items: generateFeeItems(historyResponse.purchaseHistory, { orderNotesMap }),
                boardFees: this.composeMiscFeeComponent(purchasedBatch.miscFees?.boardFees),
                universityFees: this.composeMiscFeeComponent(purchasedBatch.miscFees?.universityFees),
            };
        } catch (e: unknown) {
            // Existing error handling...
        }
    }

    // Existing return for non-purchased batches...
}

private async fetchOrderNotesForPurchaseHistory(
    purchaseHistory: PurchaseHistoryNew[]
): Promise<Map<string, Record<string, string>>> {
    const orderNotesMap = new Map<string, Record<string, string>>();
    
    // Extract order IDs from purchase history
    const orderIds = purchaseHistory
        .map(ph => ph.orderId)
        .filter(Boolean)
        .filter((id, index, arr) => arr.indexOf(id) === index); // Remove duplicates
    
    // Fetch order details for each order ID
    for (const orderId of orderIds) {
        try {
            const orderResponse = await firstValueFrom(
                this.paymentOrderService.getOrder({ id: orderId }, this.metadata)
            );
            
            // Map order notes to purchase history items with the same order ID
            purchaseHistory
                .filter(ph => ph.orderId === orderId)
                .forEach(ph => {
                    orderNotesMap.set(ph.id, orderResponse.order?.notes || {});
                });
        } catch (error) {
            this.logger.warn(`Failed to fetch order details for order ID ${orderId}:`, error);
            // Continue with other orders, don't fail the entire request
        }
    }
    
    return orderNotesMap;
}
```

### 6. Update Fee Structure DTOs

**File**: `src/modules/user/admin/student-batch/dtos/fee-structure.dto.ts`

Add cheque information to the response DTO:

```typescript
import { ChequeStatusInfo } from "../utils/cheque-status-mapper.util";

export class SDFeeComponentDto extends FeeComponent {
    // Existing properties...
    
    @ApiProperty({ 
        description: "Cheque payment information if applicable",
        required: false
    })
    chequeInfo?: ChequeStatusInfo;
}
```

### 7. Update Fee Component Entities

**File**: `src/modules/purchased-batch-fee/core/entities/fee-component.entity.ts`

```typescript
import { ChequeStatusInfo } from "@modules/user/admin/student-batch/utils/cheque-status-mapper.util";

export abstract class BaseFeeComponent {
    // Existing properties...
    chequeInfo?: ChequeStatusInfo;

    constructor(props: BaseFeeComponent) {
        // Existing constructor code...
        this.chequeInfo = props.chequeInfo;
    }
}
```

## Integration Points

### 1. Payment Status Display Logic

The SDUI composer should check the cheque status and display appropriate UI states:

- **Unclear(R)**: Show "Cheque Received" with cheque details
- **Unclear(P)**: Show "Cheque Presented" with presentation details  
- **Paid**: Show "Payment Completed" (cheque cleared)
- **Unpaid**: Show "Payment Pending" (cheque rejected/bounced/returned)

### 2. Fee Structure Breakup

The fee breakup should reflect the cheque status in the payment status field, allowing the frontend to display appropriate indicators and actions.

### 3. Cheque Details Display

When cheque information is available, include:
- Cheque number and date
- Current status and last update timestamp
- Documents (received/presented receipts)
- Responsible admin user details

## Backward Compatibility

This implementation maintains backward compatibility by:

1. Only applying cheque logic when `manualPaymentMethod === "Cheque"`
2. Defaulting to existing payment status logic for non-cheque payments
3. Not breaking existing fee structure composition for other payment methods
4. Gracefully handling missing order notes or cheque information

## Error Handling

- Handle missing or invalid cheque status gracefully
- Log warnings for failed order detail fetches without breaking the response
- Provide fallback status values for incomplete cheque information
- Validate cheque status transitions according to the defined flow
