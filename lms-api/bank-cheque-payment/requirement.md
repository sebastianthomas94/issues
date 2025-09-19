# Bank Cheque Payment Requirement

# Problem Statement

We have already added the bank cheque payment method in PMS. We need to integrate it with the current manual payment flow in LMS.

### Cheque Status Flow

```
1. Received → 2. Presented → 3. Cleared/Rejected/Bounced
                         ↓
                    4. Returned (back to Pending/Overdue)
```

In Our current flow the manual payment is made into a valid paid transaction at the moment of update. But in the cheque flow the cheque status goes through the above stages to reach its end of life cycle. This life cycle is handled by PMS.

This LMS feature is to integrate with the current API so that the current sdui for `students/:studentId/batches/:batchId/fees` (GET and POST) can be used to make cheque payments and update the status of the cheque.

# Implementation Requirements

- The new payment status `Unclear_R` and `Unclear_P` are added to the enum `PaymentStatus`.

```ts
export enum PaymentStatus {
  Unpaid = "Unpaid", // unpaid if cheque is not cleared(lifecycle ended with Rejected/Bounced/Returned)
  Incomplete = "Incomplete",
  Paid = "Paid", // paid if cheque is cleared
  Refunded = "Refunded",
  Cancelled = "Cancelled",
  Unclear_R = "Unclear(R)", // check has been received from the user
  Unclear_P = "Unclear(P)", // check has been presented to the bank
  Voided = "Voided",
}
```

- new `payment.proto`changes

```proto
syntax = "proto3";

package payment;
import "purchase-history.proto";
import "offer.proto";

service OrderService {
    rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);// cheque payment can be made through this method. initially the status will be "Received"
    rpc CreateDryRun(CreateOrderRequest)  returns (OrderDryRunResponse);
    rpc VerifyOrder(VerifyOrderRequest) returns (OrderResponse);
    rpc GetOrder(GetOrderRequest) returns (OrderResponse);
    rpc VerifyAppleOrder(VerifyAppleOrderRequest) returns (VerifyAppleOrderResponse);
    rpc UpdateChequeStatus(UpdateChequeStatusRequest) returns (UpdateChequeStatusResponse); // newly added method
}

// Enums

enum CurrencyType {
    INR = 0;
}

enum OrderStatus {
    Created = 0;
    Attempted = 1;
    Verified = 2;
    Captured = 3;
    Failed = 4;
}


enum PaymentMedium {
    Razorpay = 0;
    ApplePay = 1;
    Manual = 2;
}

enum ManualPaymentMethod {
    UPI = 0;
    Bank = 1;
    Cash = 2;
    Card = 3;
    Cheque = 4;
}

enum ChequeStatus {
    ChequeReceived = 0;
    ChequePresented = 1;
    ChequeCleared = 2;
    ChequeRejected = 3;
    ChequeBounced = 4;
    ChequeReturned = 5;
}

// Messages

message Customer {
    string id = 1;
    string name = 2;
    string contact = 4;
    optional string email = 3;
    optional string address = 5;
}

message DocumentDetails {
    string document_hash = 1;
    string document_asset_url = 2;
    string document_name = 3;
}

message DocumentDetailsList {
    repeated DocumentDetails documents = 1;
}

message ManualPaymentDetails {
     oneof optional_documents {
        DocumentDetailsList document_details = 1;
    }
    optional string created_by = 2;
    optional string creator_number = 3;
    optional string manual_payment_id = 4;
    optional ManualPaymentMethod manual_payment_method = 5;
    optional string invoice_id = 6;
    optional string cheque_number = 7;
    optional string cheque_date = 8;
}

message CreateOrderRequest {
    int32 amount = 2;
    CurrencyType currency = 3;
    string receipt = 4;
    map<string, string> notes = 5;
    Customer customer = 6;
    string referred_by = 7;
    PaymentMedium medium = 8;
    string batch_allocation_id = 9;
    ManualPaymentDetails manual_payment_details = 10;
    string batch_name = 11;
    string course_name = 12;
    repeated string offer_ids = 13;
    repeated string skip_component_ids = 14;
}



message VerifyOrderRequest {
    string payment_id = 1;
    string order_id = 2;
    string signature = 3;
}


message VerifyAppleOrderRequest {
    string transaction_id = 1;
}


message VerifyAppleOrderResponse {
    string message = 1;
}


message OrderResponse{
    string id = 1;
    int32 amount = 2;
    int32 amount_paid = 3;
    CurrencyType currency = 4;
    OrderStatus status = 5;
    map<string, string> notes = 6;
    string created_at = 7;
    int32 amount_paid_in_paise = 8;
    optional string payment_id = 9;
    bool is_captured = 10;
    string receipt = 11;
    Customer customer = 12;
    string batch_allocation_id = 13;
}

message CreateOrderResponse {
    OrderResponse order = 1;
    string alert_message = 2;
}

message GetOrderRequest {
    string id = 1;
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

message OrderDryRunResponse {
     repeated batches.PurchaseHistoryNew purchase_history = 1;
     repeated batches.Offer used_offers = 2;
}




```

- The current cheque status will be inside order.notes
  `example order.notes`

```json
{
  "source": "website",
  "purpose": "Course enrollment",
  "offerIds": ["offer_abc123", "offer_def456"],
  "referred": "referral_789",
  "chequeDate": "2025-09-16",
  "customerId": "YHwNLS",
  "chequeNumber": "CHQ123456",
  "chequeStatus": "Presented", // status
  "lastUpdatedBy": "admin_user",
  "manualInvoiceId": "inv_556677",
  "manualPaymentId": "pay_9876512",
  "presentedReason": "Cheque cleared successfully",
  "lastStatusUpdate": "2025-09-17T10:14:19.255Z",
  "presentationDate": "2025-09-16T10:00:00Z",
  "skipComponentIds": ["comp_101", "comp_202"],
  "receivedDocuments": [
    {
      "documentHash": "hash_ABC123",
      "documentAssetUrl": "https://example.com/cheques/CHQ123456.jpg",
      "documentName": "ChequeImage.jpg"
    }
  ],
  "presentedDocuments": [
    {
      "documentHash": "hash_XYZ123",
      "documentAssetUrl": "https://example.com/receipts/CHQ123456-cleared.pdf",
      "documentName": "ClearedChequeReceipt.pdf"
    }
  ],
  "manualPaymentMethod": "Cheque",
  "manualOrderCreatedBy": "admin_user",
  "manualOrderCreatorNumber": "+919999999999"
}
```

- The sdui composer will need to check the cheque status and send the appropriate payment status enum to the frontend.
