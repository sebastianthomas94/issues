# Check Payment Requirement

We are adding new feature of manual payment method as bank cheque.

```ts
export enum ManualPaymentMethodStr {
  UPI = "UPI",
  Bank = "Bank",
  Cash = "Cash",
  Card = "Card",
  Cheque = "Cheque",
  UNRECOGNIZED = "Unrecognized",
}
```

### **Requirement**

- Only installments are allowed to be paid by cheque.
- if the cheque date is greater than current due date the due date is extended to cheque date. if not due date remains same.

### Flow

Unlike other manual payment for this payment there would be steps of verification. The flow would be like this:

1. User pays for the order in cheque. The cheque details like amount, cheque number, date and a photo of cheque is taken.
   - The Order status will be "Received". Meaning cheque has been received but not cleared.

```ts
export enum ChequeStatus {
  Received = "Received", // cheque received from the user
  Presented = "Presented", // cheque presented to bank
  Returned = "Returned",
  Cleared = "Cleared",
  Rejected = "Rejected",
  Bounced = "Bounced",
}
```

2. Admin presents the cheque to bank. The status is updated to "Presented" or "Returned"(by the bank). The date will be recorded along with the photo of the receipt.
   - If returned the status will go back to pending of overdue(if the due date is passed).

```ts
export enum PurchaseHistoryStatusStr {
  Pending = "Pending",
  Completed = "Completed",
  Overdue = "Overdue",
  UNRECOGNIZED = "UNRECOGNIZED",
}
```

3. Admin will verify if the cheque is cleared or rejected or bounced. The date and receipt photo will be recorded.
   - If cleared the status will be "Completed".
   - If rejected or bounced the status will be "Overdue" or "Pending" based on due date.




## Implementation

- There won't be a need of new route or controller.
- The orders will be created like always with OrderController.createOrder.
- The order notes will be appended with the cheque details. if the payment method is cheque, the order notes will always have cheque status.
- In order line the payment method will be ManualPaymentMethodStr.Cheque.
- In every update the check status will change to its progressive or regressive state.


INSTRUCTION: Go through the code and create a concise implementation documentation for github issue (cheque-feat-github-issue-v2.md). Do not need to be very elaborate. Just the key points, challenges and implementation details.
**DO NOT MAKE ANY OTHER EDITS IN THE CODE.**
