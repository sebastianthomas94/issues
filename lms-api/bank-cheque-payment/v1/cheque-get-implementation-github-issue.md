# [Feature] Integrate Cheque Payment Status Display in Fee Structure GET API

## 📋 Summary
Integrate bank cheque payment status display in the existing `GET /api/v1/x/students/:studentId/batches/:batchId/fees` endpoint to support the cheque payment lifecycle (Received → Presented → Cleared/Rejected/Bounced/Returned).

## 🎯 Requirements
- Display cheque payment status in existing fee structure response
- Support new payment statuses: `Unclear(R)` and `Unclear(P)`
- Show cheque details (number, date, documents, status history)
- Maintain backward compatibility with existing payment methods

## 📝 Implementation Tasks

### 1. Update PaymentStatus Enum
**File:** `src/modules/purchased-batch-fee/core/enums/fee-payment-status.enum.ts`
```diff
export enum PaymentStatus {
    Unpaid = "Unpaid",
    Incomplete = "Incomplete",
    Paid = "Paid",
    Refunded = "Refunded",
    Cancelled = "Cancelled",
+   Unclear_R = "Unclear(R)", // cheque received
+   Unclear_P = "Unclear(P)", // cheque presented
    Voided = "Voided",
}
```

### 2. Create Cheque Status Mapper
**File:** `src/modules/user/admin/student-batch/utils/cheque-status-mapper.util.ts`
- `mapChequeStatusToPaymentStatus()` - Maps cheque lifecycle to PaymentStatus
- `extractChequeInfo()` - Extracts cheque details from order notes
- `ChequeStatusInfo` interface for typed cheque data

### 3. Enhance Response DTO
**File:** `src/modules/user/admin/student-batch/dtos/student-batch.dto.ts`
```diff
export class GetFeeStructureResponseDto {
    feeStructures: SDFeeStructureDto[];
    paymentDetails?: PaymentPart[];
    items: FeeItem[];
    boardFees: (MiscFeeComponentDto | PartialMiscFeeComponentDto)[];
    universityFees: (MiscFeeComponentDto | PartialMiscFeeComponentDto)[];
+   chequeInfo?: Record<string, ChequeStatusInfo>; // Parallel cheque data
}
```

### 4. Update Student Batch Service
**File:** `src/modules/user/admin/student-batch/services/student-batch.service.ts`
- Add `fetchOrderNotesForPurchaseHistory()` method
- Add `extractChequeInfoForComponents()` method  
- Enhance `getFeeStructure()` to include cheque information

## 🔄 Cheque Status Flow
```
Received (Unclear_R) → Presented (Unclear_P) → Cleared (Paid)
                                           ↘ Rejected/Bounced/Returned (Unpaid)
```

## 📊 API Response Structure
```json
{
  "feeStructures": [...], // Existing structure unchanged
  "paymentDetails": [...],
  "items": [...],
  "chequeInfo": {
    "component_id_123": {
      "chequeStatus": "Presented",
      "chequeNumber": "CHQ123456", 
      "chequeDate": "2025-09-16",
      "presentationDate": "2025-09-17",
      "lastUpdatedBy": "admin_user",
      "lastStatusUpdate": "2025-09-17T10:14:19.255Z",
      "receivedDocuments": [...],
      "presentedDocuments": [...]
    }
  },
  "boardFees": [...],
  "universityFees": [...]
}
```

## ✅ Acceptance Criteria
- [ ] New PaymentStatus enum values are supported
- [ ] Cheque payment status is correctly mapped and displayed
- [ ] Cheque details (number, date, documents) are included in response
- [ ] Non-cheque payments continue to work unchanged
- [ ] Error handling for missing/invalid cheque data
- [ ] Backward compatibility maintained

## 🔧 Technical Notes
- Uses **parallel information approach** - no core SDUI component changes required
- Cheque data fetched from PMS order notes via `getOrder` RPC
- Frontend can merge `chequeInfo[componentId]` with fee components
- Graceful degradation when order details are unavailable

## 🧪 Testing Requirements
- [ ] Unit tests for cheque status mapping utilities
- [ ] Integration tests for enhanced service methods
- [ ] E2E tests for complete cheque payment flow
- [ ] Backward compatibility tests for existing payment methods

## 📚 Related
- Depends on: PMS cheque payment integration
- Related to: POST endpoint for cheque status updates
- Documentation: Update API docs with new response structure

## 🏷️ Labels
`feature` `payment` `cheque` `api` `backend` `sdui`
