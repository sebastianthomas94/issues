# Cheque Payment Feature Implementation

## Overview
This document outlines the implementation details for the manual payment method "Cheque" feature, including requirements, challenges, and technical implementation considerations.

## Feature Requirements

### Core Requirements
- **Restriction**: Only installments are allowed to be paid by cheque (not one-time payments)
- **Due Date Extension**: If cheque date > current due date, extend due date to cheque date; otherwise keep existing due date
- **Multi-Step Verification Flow**: Cheque payments require progressive status updates through verification steps

### Cheque Status Flow
```
1. Received → 2. Presented → 3. Cleared/Rejected/Bounced
                         ↓
                    4. Returned (back to Pending/Overdue)
```

### Status Mappings
- **Received** → Order Status: "Received" (custom status)
- **Presented** → Status remains "Received" 
- **Returned** → Order Status: "Pending" or "Overdue" (based on due date)
- **Cleared** → Order Status: "Completed"
- **Rejected/Bounced** → Order Status: "Overdue" or "Pending" (based on due date)

## Current Architecture Analysis

### 1. Enum Structure
**Current ManualPaymentMethod (proto/payment.proto)**:
```proto
enum ManualPaymentMethod {
    UPI = 0;
    Bank = 1;
    Cash = 2;
    Card = 3;
}
```
**Required**: Add `Cheque = 4` to the enum

**Current ManualPaymentMethodStr (src/modules/funds/orders/enums/order.enum.ts)**:
```typescript
export enum ManualPaymentMethodStr {
    UPI = "UPI",
    Bank = "Bank", 
    Cash = "Cash",
    Card = "Card",
    UNRECOGNIZED = "Unrecognized",
}
```
**Required**: Add `Cheque = "Cheque"`

### 2. Order Notes Structure
**Current Implementation**: Order notes store manual payment details as key-value pairs:
- `manualPaymentMethod`: Payment method enum value
- `manualPaymentId`: Unique payment identifier  
- `manualOrderCreatedBy`: Creator information
- `manualDocumentDetails`: Document information (JSON)

**Required for Cheque**: Additional notes fields:
- `chequeStatus`: Current cheque verification status
- `chequeNumber`: Cheque number
- `chequeDate`: Cheque date
- `chequeAmount`: Cheque amount
- `presentedDate`: Date when presented to bank
- `statusUpdateHistory`: JSON array of status changes with timestamps

### 3. Order Creation Flow
**Current Flow** (OrderController.createOrder):
```
1. CreateOrderDto validation
2. ManualOrderStrategy.create()
3. ManualOrdersHelper.create() 
4. Order notes populated via handleManualOrderNotes()
5. Order persisted with status "Created"
6. Manual transaction event emitted
```

**Required Changes**:
- **Validation**: Ensure cheque payments only allow instalment-based orders
- **Due Date Logic**: Implement due date extension logic in order line creation
- **Status Handling**: Set initial status to "Received" for cheque payments

### 4. Instalment Payment Processing
**Current Implementation**: 
- Order lines created via `createOrderLineWithNewPurchaseHistory()`
- Instalments processed individually with `PaymentTypeFundsStr.InstalmentFull/InstalmentPartial`
- Due dates maintained in instalment entities

**Required Logic**:
```typescript
// In order line creation
if (paymentMethod === ManualPaymentMethodStr.Cheque) {
    // Only allow instalment payments
    if (isOneTimePayment) {
        throw new RpcInvalidArgumentException("Cheque payments only allowed for instalments");
    }
    
    // Due date extension logic
    const chequeDate = new Date(createOrder.notes.chequeDate);
    const currentDueDate = new Date(instalment.dueAt);
    
    if (chequeDate > currentDueDate) {
        // Extend due date to cheque date
        instalment.dueAt = chequeDate;
        // Record due date change in component history
    }
}
```

### 5. Status Update Mechanism
**Current Manual Transaction Flow**:
```
ManualOrdersHelper.processPostOrderCreation() → 
EventEmitter.emit("transaction.create") →
CreateManualTransactionStrategy.create() →
Order status updated to "Verified"/"Failed"
```

**Required for Cheque Multi-Step Flow**:
- **Custom Update Endpoints**: New API endpoints for status updates
- **Status Validation**: Ensure proper status progression 
- **Document Handling**: Store receipts/photos for each step
- **Date Tracking**: Record timestamps for each status change

## Implementation Challenges

### 1. **Instalment-Only Restriction**
**Challenge**: Preventing cheque payments for one-time payments while maintaining existing order flow

**Solution**: Add validation in `ManualOrderStrategy.checkIfRequestMatchForOrder()`:
```typescript
private checkChequePaymentRestrictions(createOrder: ICreateOrder) {
    if (createOrder.manualPaymentDetails?.manualPaymentMethod === ManualPaymentMethodStr.Cheque) {
        const isOneTimePayment = this.determinePaymentType(createOrder);
        if (isOneTimePayment) {
            throw new RpcInvalidArgumentException("Cheque payments are only allowed for instalment-based orders");
        }
    }
}
```

### 2. **Due Date Extension Logic**
**Challenge**: Modifying instalment due dates during order creation without breaking existing logic

**Solution**: Extend `createOrderLineWithNewPurchaseHistory()` utility:
```typescript
// In createOrderLineWithNewPurchaseHistory()
if (order.notes.manualPaymentMethod === ManualPaymentMethodStr.Cheque) {
    const chequeDate = new Date(order.notes.chequeDate);
    const originalDueDate = new Date(instalment.dueAt);
    
    if (chequeDate > originalDueDate) {
        instalment.dueAt = chequeDate;
        // Create component history record for due date change
        this.recordDueDateChange(instalment.id, originalDueDate, chequeDate, "Cheque date extension");
    }
}
```

### 3. **Multi-Step Status Management**
**Challenge**: Current order system supports simple Created→Verified/Failed flow; cheque requires intermediate states

**Considerations**:
- **Option A**: Extend existing OrderStatus enum (requires proto changes)
- **Option B**: Use order notes to track cheque status (minimal changes)
- **Recommendation**: Option B for faster implementation

### 4. **Status Progression Validation**
**Challenge**: Ensuring valid status transitions and preventing invalid updates

**Solution**: Status transition validation:
```typescript
const validTransitions = {
    [ChequeStatus.Received]: [ChequeStatus.Presented, ChequeStatus.Returned],
    [ChequeStatus.Presented]: [ChequeStatus.Cleared, ChequeStatus.Rejected, ChequeStatus.Bounced],
    [ChequeStatus.Returned]: [ChequeStatus.Presented],
    // Terminal states
    [ChequeStatus.Cleared]: [],
    [ChequeStatus.Rejected]: [],
    [ChequeStatus.Bounced]: []
};
```

### 5. **Purchase History Status Mapping**
**Challenge**: Mapping cheque statuses to PurchaseHistoryStatus for consistent reporting

**Solution**: Status mapping logic:
```typescript
const mapChequeStatusToPurchaseStatus = (chequeStatus: ChequeStatus, dueDate: Date) => {
    switch (chequeStatus) {
        case ChequeStatus.Cleared:
            return PurchaseHistoryStatusStr.Completed;
        case ChequeStatus.Received:
        case ChequeStatus.Presented:
            return new Date() > dueDate ? PurchaseHistoryStatusStr.Overdue : PurchaseHistoryStatusStr.Pending;
        case ChequeStatus.Rejected:
        case ChequeStatus.Bounced:
        case ChequeStatus.Returned:
            return new Date() > dueDate ? PurchaseHistoryStatusStr.Overdue : PurchaseHistoryStatusStr.Pending;
    }
};
```

## Key Implementation Points

### 1. **Proto Updates Required**
- Add `Cheque = 4` to `ManualPaymentMethod` enum
- Regenerate proto TypeScript files
- Update mapping constants in `order-map.constant.ts`

### 2. **Order Notes Extensions**
- Extend `handleManualOrderNotes()` to store cheque-specific data
- Implement cheque status tracking in notes
- Add validation for required cheque fields (number, date, amount)

### 3. **Instalment Integration**
- Modify order line creation utilities for due date extension
- Integrate with existing component history tracking
- Ensure compatibility with existing instalment update flows

### 4. **Status Update API**
- Create dedicated endpoints for cheque status updates
- Implement proper authorization and validation
- Add audit trail for all status changes

### 5. **Testing Considerations**
- Unit tests for instalment-only validation
- Integration tests for due date extension logic  
- End-to-end tests for complete status flow
- Edge case testing (multiple instalments, overlapping due dates)

## Conclusion

The cheque payment feature can be implemented within the existing architecture with targeted modifications to:
1. Enum definitions and mappings
2. Order creation validation logic
3. Instalment due date handling
4. Status tracking via order notes
5. Purchase history status computation

The modular design of the current payment system allows for these extensions without major architectural changes, following the established patterns for manual payment processing.
