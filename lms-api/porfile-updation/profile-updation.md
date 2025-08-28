# User Profile & Admission Form Separation

## Problem Statement

Currently, the admission form database can be modified by users after submission. This creates data integrity issues as admission details should remain immutable once submitted, even if user information becomes outdated.

## Current Issues

- Users can modify their admission form data post-submission
- Admission records lose historical accuracy when user details change
- No clear separation between mutable user profile and immutable admission records

## Proposed Solution

### Schema Changes

1. **Make Admission Forms Immutable**: Once created, admission form records cannot be modified
2. **Add Mutable User Details**: Introduce a new `details` field in the User schema for user-editable information

### New User Schema Structure

```ts
export class User {
  // ...existing fields...
  details: UserDetails; // New mutable field
}

class UserDetails {
  dateOfBirth: Date;
  gender: Gender;
  whatsapp: string;
  email: string;
  address: Address;
  studentQualification: StudentQualification;
  board: EducationBoard;
  instituteAddress: string;
  stream: EducationStream;
  guardianName: string;
  guardianPhone: string;
}
```

### Implementation Strategy

1. **Event-Driven Updates**: When admission forms are submitted, emit an event to copy data to user details
2. **Route Separation**: 
   - `PATCH /v2/users/me` - Updates only the mutable `details` field
   - Maintain v1 routes for backward compatibility
3. **Data Flow**: Admission form → Event → User details (overwrite if exists)

### Benefits

- **Data Integrity**: Admission records remain unchanged for audit purposes
- **User Flexibility**: Users can update their current information without affecting admission history
- **Backward Compatibility**: v1 API continues to work while v2 provides new functionality
- **Clear Separation**: Distinct boundaries between immutable admission data and mutable user profile
