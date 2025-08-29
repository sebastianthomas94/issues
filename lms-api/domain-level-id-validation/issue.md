# Problem Statement

In various sections of our API, there are DTOs which accept IDs of domain entities like courseId, stageId, paperId, facultyId and so on. Although there are DTO-level validations present for type checking, these IDs are not validated beyond that at the database level. This creates a risk where users can enter invalid IDs (by calling the API directly), leading to an invalid state in the DB. In some cases, such attempts result in a 500 error, which is also inappropriate.

The affected endpoints that have caught my attention are:

- Batch Content (Creation & Update)
- Paper Service (Creation & Update)
- Template Service (Creation & Update)

# Solution

## 1. Propraitroy Sevice for Domain Level ID Validation

Introduce a domain validator service specifically for validating entity IDs. This service should inject all relevant domain services (e.g., Course, Stage, Paper, etc.) and provide simple validation methods like validateCourseAndStage(courseId, stageId) for use across all services. Care should be taken to avoid circular dependencies (for example, if the Course service injects the validator to validate Templates), which could be mitigated by using forwardRef where necessary.

## 2. DTO-Level Validation

We fail first at the DTO level using class-validator decorators. This ensures that invalid IDs are caught early in the request lifecycle, preventing unnecessary processing and potential errors down the line.

`ensure-exists.validator.ts`

```ts
@ValidatorConstraint({ async: true })
@Injectable()
export class EnsureExistsConstraint implements ValidatorConstraintInterface {
  constructor(private readonly moduleRef: ModuleRef) {}

  async validate(value: any, args: ValidationArguments) {
    const [repoToken] = args.constraints;
    // Resolve the repository from Nest's DI container
    const repo = this.moduleRef.get(repoToken, { strict: false });

    if (!repo || typeof repo.findById !== "function") {
      throw new Error(
        `Invalid repository for @EnsureExists: ${repoToken?.toString()}`
      );
    }

    const entity = await repo.findById(value);
    return !!entity;
  }

  defaultMessage(args: ValidationArguments) {
    return `${args.property} with value ${args.value} does not exist`;
  }
}

export function EnsureExists(
  repoToken: any,
  validationOptions?: ValidationOptions
) {
  return function (object: Object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName,
      options: validationOptions,
      constraints: [repoToken],
      validator: EnsureExistsConstraint,
    });
  };
}
```

`exapmle.dto.ts`

```ts
export class CreatePaperDto {
  // existing fields...
  @IsString()
  @IsNotEmpty()
  @EnsureExists(CourseRepository)
  courseId: string;

  @IsString()
  @IsNotEmpty()
  @EnsureExists(StageRepository)
  stageId: string;
}
```

`main.ts`

```ts
// existing code...

useContainer(app.select(AppModule), { fallbackOnErrors: true });
// existing code...
```

`EnsureExistsConstraint` must be a global provider to be resolved in DTOs.<!--  -->
