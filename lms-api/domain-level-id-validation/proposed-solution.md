## DTO-Level Validation with `@EnsureExists`
Custom class-validator decorator that validates entity existence at the DTO level:

```typescript
export class CreatePaperDto {
  @IsString()
  @IsNotEmpty()
  @EnsureExists(CourseRepository)
  courseId: string;

  @EnsureExists(StageRepository)
  stageId: string;
}
```

The `EnsureExistsConstraint` uses Nest's `ModuleRef` to resolve repositories dynamically and check entity existence, providing clear validation messages.

### Implementation Requirements
- Register `EnsureExistsConstraint` as global provider
- Enable class-validator container integration in `main.ts`
- Update affected DTOs with `@EnsureExists` decorators
