## **IMMUTABLE SCHEMAS (Cannot be modified after creation):**

### 1. **Transaction-related (Financial Records)**
- ✅ `TransactionEntity` (`transactions`) - No UpdateDateColumn, financial audit trail
- ✅ `TransactionLineEntity` (`transaction_lines`) - ALL fields have `update: false`
- ✅ `BillingDocumentEntity` (`billing_documents`) - No UpdateDateColumn, legal documents

### 2. **Order Line Items (Immutable Financial Records)**
- ✅ `OrderLineEntity` (`order_lines`) - Most fields have `update: false`

### 3. **Fee Components (Base Templates)**
- ✅ `FeeComponentEntity` (`fee_components`) - Core fields have `update: false`

### 4. **Wallet Ledger (Audit Trail)**
- ✅ `WalletLedgerEntity` (`wallet_ledgers`) - No UpdateDateColumn, financial ledger

### 5. **Offers (Once Created)**
- ✅ `OfferEntity` (`offers`) - No UpdateDateColumn

### 6. **Database Views (Read-only)**
- ✅ `TotalWithdrawalsViewEntity` - Uses `@ViewEntity`
- ✅ `ReferralAnalyticsViewEntity` - Uses `@ViewEntity`
- ✅ `FeeComponentViewEntity` - Uses `@ViewEntity`
- ✅ `InstalmentViewEntity` - Uses `@ViewEntity`
- ✅ `OfferWithStatusViewEntity` - Uses `@ViewEntity`
- ✅ `PurchaseHistoryViewEntity` - Uses `@ViewEntity`

## **MUTABLE SCHEMAS (Can be updated):**

### 1. **User/Business Data**
- 🔄 `OrderEntity` (`orders`) - Has UpdateDateColumn, status can change
- 🔄 `PaymentAccountEntity` (`payment_accounts`) - Has UpdateDateColumn
- 🔄 `WalletRecordEntity` (`wallet_records`) - Has UpdateDateColumn

### 2. **Process Management**
- 🔄 `BatchAllocationEntity` (`batch_allocations`) - Has UpdateDateColumn, status changes
- 🔄 `ReferralEntity` (`referrals`) - Has UpdateDateColumn, status updates
- 🔄 `ReferralWithdrawalEntity` (`referral_withdrawals`) - Has UpdateDateColumn

### 3. **Conditional Immutability**
- 🔄 `InstalmentEntity` (`instalments`) - Has `is_immutable` flag for conditional immutability