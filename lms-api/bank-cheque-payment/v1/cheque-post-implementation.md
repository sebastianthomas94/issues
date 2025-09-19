# Cheque Payment POST Implementation

## Overview
This document outlines the implementation for updating cheque payment status through the existing `POST /api/v1/x/students/:studentId/batches/:batchId/fees` endpoint.

## Changes Required

### 1. Update Manual Payment DTO

**File**: `src/modules/user/admin/student-batch/dtos/student-batch.dto.ts`

Extend the existing DTO to support cheque payments:

```typescript
export class AddManualFeePaymentDto {
    @IsBoolean()
    isDryRun: boolean;

    @IsNotEmpty()
    amountBeingPaid: number;

    @IsNotEmpty()
    paymentMethod: FeePaymentType;

    @IsString()
    @IsNotEmpty()
    transactionReferenceId: string;

    // New cheque-specific fields
    @ApiProperty({ 
        description: "Cheque number (required for cheque payments)",
        required: false 
    })
    @IsOptional()
    @IsString()
    chequeNumber?: string;

    @ApiProperty({ 
        description: "Cheque date in YYYY-MM-DD format (required for cheque payments)",
        required: false 
    })
    @IsOptional()
    @IsDateString()
    chequeDate?: string;

    @ApiProperty({ 
        description: "Document details for cheque image/receipt",
        required: false 
    })
    @IsOptional()
    @ValidateNested()
    @Type(() => DocumentDetailsList)
    chequeDocuments?: DocumentDetailsList;
}
```

### 2. Create Cheque Payment Validation

**File**: `src/modules/user/admin/student-batch/validators/cheque-payment.validator.ts`

```typescript
import { BadRequestException } from "@nestjs/common";
import { FeePaymentType } from "../enums/fee-payment-type.enum";
import { AddManualFeePaymentDto } from "../dtos/student-batch.dto";

export function validateChequePaymentData(dto: AddManualFeePaymentDto): void {
    if (dto.paymentMethod === FeePaymentType.Cheque) {
        if (!dto.chequeNumber || dto.chequeNumber.trim() === "") {
            throw new BadRequestException("Cheque number is required for cheque payments");
        }
        
        if (!dto.chequeDate) {
            throw new BadRequestException("Cheque date is required for cheque payments");
        }
        
        // Validate cheque date is not in future
        const chequeDate = new Date(dto.chequeDate);
        const today = new Date();
        today.setHours(23, 59, 59, 999); // End of today
        
        if (chequeDate > today) {
            throw new BadRequestException("Cheque date cannot be in the future");
        }
        
        // Validate cheque date is not too old (e.g., more than 6 months)
        const sixMonthsAgo = new Date();
        sixMonthsAgo.setMonth(sixMonthsAgo.getMonth() - 6);
        
        if (chequeDate < sixMonthsAgo) {
            throw new BadRequestException("Cheque date cannot be more than 6 months old");
        }
    }
}
```

### 3. Update Fee Payment Type Enum

**File**: `src/modules/user/admin/student-batch/enums/fee-payment-type.enum.ts`

```typescript
export enum FeePaymentType {
    Cash = "Cash",
    Card = "Card", 
    Bank = "Bank",
    NetBanking = "NetBanking",
    UPI = "UPI",
    Cheque = "Cheque", // New cheque payment type
}
```

### 4. Create Cheque Update DTO

**File**: `src/modules/user/admin/student-batch/dtos/cheque-update.dto.ts`

```typescript
import { ApiProperty } from "@nestjs/swagger";
import { IsEnum, IsOptional, IsString, IsDateString, ValidateNested, Type } from "class-validator";
import { DocumentDetailsList } from "@modules/user/student/submodules/payment/dtos/payment-manual-order.dto";

export enum ChequeStatusUpdate {
    Present = "Present",
    Clear = "Clear", 
    Reject = "Reject",
    Bounce = "Bounce",
    Return = "Return",
}

export class UpdateChequeStatusDto {
    @ApiProperty({ 
        description: "New cheque status",
        enum: ChequeStatusUpdate 
    })
    @IsEnum(ChequeStatusUpdate)
    chequeStatus: ChequeStatusUpdate;

    @ApiProperty({ 
        description: "Reason for status update",
        required: false 
    })
    @IsOptional()
    @IsString()
    reason?: string;

    @ApiProperty({ 
        description: "Presentation date (required for Present status)",
        required: false 
    })
    @IsOptional()
    @IsDateString()
    presentationDate?: string;

    @ApiProperty({ 
        description: "Clearance date (required for Clear status)",
        required: false 
    })
    @IsOptional()
    @IsDateString()
    clearanceDate?: string;

    @ApiProperty({ 
        description: "Supporting documents (receipts, etc.)",
        required: false 
    })
    @IsOptional()
    @ValidateNested()
    @Type(() => DocumentDetailsList)
    receiptDocuments?: DocumentDetailsList;

    @ApiProperty({ 
        description: "Admin user making the update",
        required: true 
    })
    @IsString()
    updatedBy: string;
}

export class UpdateChequeStatusResDto {
    @ApiProperty({ description: "Success message" })
    message: string;

    @ApiProperty({ description: "Updated cheque status" })
    chequeStatus: string;

    @ApiProperty({ description: "Updated payment status" })
    paymentStatus: string;
}
```

### 5. Update Student Batch Service

**File**: `src/modules/user/admin/student-batch/services/student-batch.service.ts`

Add cheque payment support and status update methods:

```typescript
import { validateChequePaymentData } from "../validators/cheque-payment.validator";
import { UpdateChequeStatusDto, UpdateChequeStatusResDto, ChequeStatusUpdate } from "../dtos/cheque-update.dto";
import { UpdateChequeStatusRequest, ChequeStatus } from "proto/payment";

async addManualFeePayment(
    studentId: string,
    batchId: string,
    dto: AddManualFeePaymentDto,
): Promise<AddManualFeePaymentResDto> {
    this.logger.log(
        `Adding manual fee payment for student ${studentId} in batch ${batchId}. Amount: ${dto.amountBeingPaid}, Method: ${dto.paymentMethod}, Dry run: ${dto.isDryRun}`,
    );

    // Validate cheque payment data if applicable
    validateChequePaymentData(dto);

    // Validate and get necessary data
    const batch = await this.readBatchService.findById(batchId);
    const purchasedBatch = await this.purchasedBatchService.findOneByBatchIdAndStudentId(batchId, studentId);

    if (!purchasedBatch) {
        throw new NotFoundException(`No purchased batch found for student ${studentId} and batch ${batchId}`);
    }

    if (!purchasedBatch.batchAllocationId) {
        throw new InternalServerErrorException("Batch allocation ID is missing for purchased batch");
    }

    // Get student details for the payment order
    const student = await this.studentService.findById(studentId);

    // Create enhanced order request with cheque information
    const createOrderRequest: CreateOrderRequest = {
        amount: formatAsPaisa(dto.amountBeingPaid),
        currency: CurrencyType.INR,
        receipt: batchId,
        notes: this.buildOrderNotes(dto, studentId, batchId),
        customer: {
            id: student.id,
            name: student.name,
            contact: student.phone,
            email: student.email,
        },
        referredBy: "",
        medium: PaymentMedium.Manual,
        batchAllocationId: purchasedBatch.batchAllocationId,
        manualPaymentDetails: {
            manualPaymentId: dto.transactionReferenceId,
            manualPaymentMethod: mapPaymentMethod(dto.paymentMethod),
            chequeNumber: dto.chequeNumber,
            chequeDate: dto.chequeDate,
            documentDetails: dto.chequeDocuments,
        },
        batchName: batch.title,
        courseName: batch.title,
        offerIds: [],
        skipComponentIds: [],
    };

    // Rest of existing implementation...
}

private buildOrderNotes(
    dto: AddManualFeePaymentDto,
    studentId: string,
    batchId: string
): Record<string, string> {
    const notes: Record<string, string> = {
        batchId: batchId,
        studentId: studentId,
        paymentType: "manual",
        manualPaymentMethod: dto.paymentMethod,
    };

    // Add cheque-specific notes
    if (dto.paymentMethod === FeePaymentType.Cheque) {
        notes.chequeNumber = dto.chequeNumber!;
        notes.chequeDate = dto.chequeDate!;
        notes.chequeStatus = "Received"; // Initial status
        notes.lastStatusUpdate = new Date().toISOString();
        notes.lastUpdatedBy = "system"; // Could be extracted from auth context
        
        if (dto.chequeDocuments?.documents) {
            notes.receivedDocuments = JSON.stringify(dto.chequeDocuments.documents);
        }
    }

    return notes;
}

async updateChequeStatus(
    studentId: string,
    batchId: string,
    orderId: string,
    dto: UpdateChequeStatusDto,
): Promise<UpdateChequeStatusResDto> {
    this.logger.log(
        `Updating cheque status for student ${studentId}, batch ${batchId}, order ${orderId} to ${dto.chequeStatus}`
    );

    // Validate status transition logic
    this.validateChequeStatusTransition(dto);

    try {
        // Call PMS to update cheque status
        const updateRequest: UpdateChequeStatusRequest = {
            orderId: orderId,
            chequeStatus: this.mapToProtoChequeStatus(dto.chequeStatus),
            updatedBy: dto.updatedBy,
            reason: dto.reason,
            receiptDocuments: dto.receiptDocuments,
            presentationDate: dto.presentationDate,
            clearanceDate: dto.clearanceDate,
        };

        const response = await firstValueFrom(
            this.paymentOrderService.updateChequeStatus(updateRequest, this.metadata)
        );

        // Determine the new payment status based on cheque status
        const paymentStatus = this.mapChequeStatusToPaymentStatus(dto.chequeStatus);

        return {
            message: response.message,
            chequeStatus: dto.chequeStatus,
            paymentStatus: paymentStatus,
        };
    } catch (error) {
        this.logger.error(
            `Failed to update cheque status for order ${orderId}: ${error.message}`
        );
        throw new InternalServerErrorException("Failed to update cheque status");
    }
}

private validateChequeStatusTransition(dto: UpdateChequeStatusDto): void {
    switch (dto.chequeStatus) {
        case ChequeStatusUpdate.Present:
            if (!dto.presentationDate) {
                throw new BadRequestException("Presentation date is required when presenting cheque");
            }
            break;
        case ChequeStatusUpdate.Clear:
            if (!dto.clearanceDate) {
                throw new BadRequestException("Clearance date is required when clearing cheque");
            }
            break;
        case ChequeStatusUpdate.Reject:
        case ChequeStatusUpdate.Bounce:
        case ChequeStatusUpdate.Return:
            if (!dto.reason) {
                throw new BadRequestException(`Reason is required when marking cheque as ${dto.chequeStatus.toLowerCase()}`);
            }
            break;
    }
}

private mapToProtoChequeStatus(status: ChequeStatusUpdate): ChequeStatus {
    switch (status) {
        case ChequeStatusUpdate.Present:
            return ChequeStatus.ChequePresented;
        case ChequeStatusUpdate.Clear:
            return ChequeStatus.ChequeCleared;
        case ChequeStatusUpdate.Reject:
            return ChequeStatus.ChequeRejected;
        case ChequeStatusUpdate.Bounce:
            return ChequeStatus.ChequeBounced;
        case ChequeStatusUpdate.Return:
            return ChequeStatus.ChequeReturned;
        default:
            throw new BadRequestException(`Invalid cheque status: ${status}`);
    }
}

private mapChequeStatusToPaymentStatus(chequeStatus: ChequeStatusUpdate): string {
    switch (chequeStatus) {
        case ChequeStatusUpdate.Present:
            return "Unclear(P)";
        case ChequeStatusUpdate.Clear:
            return "Paid";
        case ChequeStatusUpdate.Reject:
        case ChequeStatusUpdate.Bounce:
        case ChequeStatusUpdate.Return:
            return "Unpaid";
        default:
            return "Unpaid";
    }
}
```

### 6. Update Student Batch Controller

**File**: `src/modules/user/admin/student-batch/controllers/student-batch.controller.ts`

Add the new cheque status update endpoint:

```typescript
import { UpdateChequeStatusDto, UpdateChequeStatusResDto } from "../dtos/cheque-update.dto";

@Post(":batchId/fees/orders/:orderId/cheque-status")
@UseScopes("students:write")
@ApiOperation({
    summary: "Update cheque payment status",
    description: "Updates the status of a cheque payment (present, clear, reject, bounce, return)"
})
@ApiErrorResponse([HttpStatus.BAD_REQUEST, HttpStatus.NOT_FOUND, HttpStatus.INTERNAL_SERVER_ERROR])
async updateChequeStatus(
    @Param("studentId") studentId: string,
    @Param("batchId") batchId: string,
    @Param("orderId") orderId: string,
    @Body() dto: UpdateChequeStatusDto,
): Promise<UpdateChequeStatusResDto> {
    return await this.studentBatchService.updateChequeStatus(studentId, batchId, orderId, dto);
}

// Update existing POST method to support cheque payments
@Post(":batchId/fees")
@UseScopes("students:write")
@ApiOperation({
    summary: "Add manual fee payment including cheque payments",
    description: "Process manual fee payment including support for cheque payments with initial 'Received' status"
})
async addManualFeePayment(
    @Param("studentId") studentId: string,
    @Param("batchId") batchId: string,
    @Body() dto: AddManualFeePaymentDto,
): Promise<AddManualFeePaymentResDto> {
    return await this.studentBatchService.addManualFeePayment(studentId, batchId, dto);
}
```

### 7. Update Proto Payment Types

**File**: `proto/payment.proto`

Add the new UpdateChequeStatus method and related types:

```proto
service OrderService {
    // Existing methods...
    rpc UpdateChequeStatus(UpdateChequeStatusRequest) returns (UpdateChequeStatusResponse);
}

enum ManualPaymentMethod {
    UPI = 0;
    Bank = 1;
    Cash = 2;
    Card = 3;
    Cheque = 4; // Add cheque method
}

enum ChequeStatus {
    ChequeReceived = 0;
    ChequePresented = 1;
    ChequeCleared = 2;
    ChequeRejected = 3;
    ChequeBounced = 4;
    ChequeReturned = 5;
}

message ManualPaymentDetails {
    // Existing fields...
    optional string cheque_number = 7;
    optional string cheque_date = 8;
}

message UpdateChequeStatusRequest {
    string order_id = 1;
    ChequeStatus cheque_status = 2;
    optional string updated_by = 3;
    optional string reason = 4;
    optional DocumentDetailsList receipt_documents = 5;
    optional string presentation_date = 6;
    optional string clearance_date = 7;
}

message UpdateChequeStatusResponse {
    OrderResponse order = 1;
    string message = 2;
}
```

### 8. Generate Updated Proto Files

After updating the proto file, regenerate TypeScript definitions:

```bash
pnpm build:grpc
```

### 9. Update Manual Payment Method Mapping

**File**: `src/common/utils/map-payment-method.util.ts`

```typescript
import { ManualPaymentMethod } from "proto/payment";
import { FeePaymentType } from "@modules/user/admin/student-batch/enums/fee-payment-type.enum";

export function mapPaymentMethod(paymentMethod: FeePaymentType): ManualPaymentMethod {
    switch (paymentMethod) {
        case FeePaymentType.UPI:
            return ManualPaymentMethod.UPI;
        case FeePaymentType.Bank:
            return ManualPaymentMethod.Bank;
        case FeePaymentType.Cash:
            return ManualPaymentMethod.Cash;
        case FeePaymentType.Card:
            return ManualPaymentMethod.Card;
        case FeePaymentType.Cheque:
            return ManualPaymentMethod.Cheque;
        default:
            throw new Error(`Unsupported payment method: ${paymentMethod}`);
    }
}
```

## API Endpoints

### 1. Create Cheque Payment
```
POST /api/v1/x/students/{studentId}/batches/{batchId}/fees
```

**Request Body:**
```json
{
    "isDryRun": false,
    "amountBeingPaid": 50000,
    "paymentMethod": "Cheque",
    "transactionReferenceId": "CHQ_TXN_001",
    "chequeNumber": "CHQ123456",
    "chequeDate": "2025-09-16",
    "chequeDocuments": {
        "documents": [{
            "documentHash": "hash_ABC123",
            "documentAssetUrl": "https://example.com/cheques/CHQ123456.jpg",
            "documentName": "ChequeImage.jpg"
        }]
    }
}
```

### 2. Update Cheque Status
```
POST /api/v1/x/students/{studentId}/batches/{batchId}/fees/orders/{orderId}/cheque-status
```

**Request Body:**
```json
{
    "chequeStatus": "Present",
    "presentationDate": "2025-09-17",
    "reason": "Cheque presented to bank for clearing",
    "updatedBy": "admin_user",
    "receiptDocuments": {
        "documents": [{
            "documentHash": "hash_XYZ789",
            "documentAssetUrl": "https://example.com/receipts/presentation_receipt.pdf",
            "documentName": "PresentationReceipt.pdf"
        }]
    }
}
```

## Business Logic Flow

### 1. Cheque Payment Creation
1. Validate cheque payment data (number, date, documents)
2. Create order with initial "Received" status
3. Store cheque information in order notes
4. Return payment structure with `Unclear(R)` status

### 2. Cheque Status Updates
1. **Present**: Update to `Unclear(P)`, require presentation date
2. **Clear**: Update to `Paid`, require clearance date  
3. **Reject/Bounce/Return**: Update to `Unpaid`, require reason

### 3. Status Transition Validation
- Received → Present → Clear ✅
- Received → Present → Reject/Bounce ✅
- Any status → Return ✅ (special case)

## Error Handling

### 1. Validation Errors
- Missing cheque number/date for cheque payments
- Invalid date ranges (future dates, too old)
- Invalid status transitions

### 2. Integration Errors
- PMS service unavailable
- Order not found
- Insufficient permissions

### 3. Business Rule Violations
- Attempting to update non-cheque payments
- Invalid cheque status transitions
- Missing required fields for status updates

## Backward Compatibility

- Existing manual payments continue to work unchanged
- New cheque fields are optional and only validated for cheque payments
- Existing payment status logic preserved for non-cheque payments
- API responses maintain existing structure with optional cheque information
