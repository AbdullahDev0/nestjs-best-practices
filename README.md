# NestJS: Best Practices & Robust Backend Architecture Guide
Below is an extensive list of best practices for building NestJS applications—mirroring the style and depth of the Laravel Best Practices repo you shared. This guide covers everything from applying the single responsibility principle to using DTOs and TypeORM with transactions and migrations. Each section explains the “why” and “how,” and includes code examples that follow NestJS conventions.

I have also provided a sample application flow on Medium that further explains how to build a robust backend using NestJS, TypeORM, and microservices. Check it out here:  
[Building a Robust Backend: A Comprehensive Guide Using NestJS, TypeORM, and Microservices](https://medium.com/@abdullahirfan99_80517/building-a-robust-backend-a-comprehensive-guide-using-nestjs-typeorm-and-microservices-f291295cb187)

---

## 1. Single Responsibility Principle (SRP)

**Principle:**  
Every class or function should have one responsibility only. Keep controllers, services, and repositories focused on one concern.

**Bad Example (Doing Too Much):**

```typescript
// controllers/user.controller.ts (Bad)
import { Controller, Post, Body } from '@nestjs/common';
import { CreateUserDto } from './dtos/create-user.dto';
import { User } from './entities/user.entity';

@Controller('users')
export class UserController {
  @Post()
  async create(@Body() dto: CreateUserDto): Promise<User> {
    // Validate, create user, log event, and send email in the same method
    const user = await this.someUserRepository.create(dto);
    // Log event
    console.log(`User ${user.username} created`);
    // Send email (business logic should not be here)
    await this.sendWelcomeEmail(user);
    return user;
  }
  
  // Mixed concerns!
  private async sendWelcomeEmail(user: User) {
    // ...
  }
}
```

**Good Example (One Responsibility per Class/Method):**

```typescript
// controllers/user.controller.ts (Good)
import { Controller, Post, Body } from '@nestjs/common';
import { CreateUserDto } from './dtos/create-user.dto';
import { UserService } from './user.service';
import { User } from './entities/user.entity';

@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Post()
  async create(@Body() dto: CreateUserDto): Promise<User> {
    return this.userService.createUser(dto);
  }
}
```

And in the service:

```typescript
// services/user.service.ts
import { Injectable } from '@nestjs/common';
import { CreateUserDto } from './dtos/create-user.dto';
import { User } from './entities/user.entity';

@Injectable()
export class UserService {
  async createUser(dto: CreateUserDto): Promise<User> {
    // Validate & sanitize already handled by DTO & pipes
    const user = await this.createUserRecord(dto);
    this.logUserCreation(user);
    return user;
  }

  private async createUserRecord(dto: CreateUserDto): Promise<User> {
    // Handle DB logic (e.g. repository or TypeORM)
    // ...
    return new User(); // placeholder
  }

  private logUserCreation(user: User): void {
    // Log only responsibility
    console.log(`User ${user.username} created.`);
  }
}
```

---

## 2. Methods Should Do Just One Thing

Break complex logic into small, well-named helper methods to improve readability and testability.

**Bad Example:**

```typescript
// models/user.entity.ts (Bad)
export class User {
  getDisplayName(): string {
    if (this.isClient() && this.isVerified()) {
      return `Mr. ${this.firstName} ${this.middleName} ${this.lastName}`;
    } else {
      return `${this.firstName[0]}. ${this.lastName}`;
    }
  }
}
```

**Good Example:**

```typescript
// models/user.entity.ts (Good)
export class User {
  getDisplayName(): string {
    return this.isVerifiedClient() ? this.getLongName() : this.getShortName();
  }

  private isVerifiedClient(): boolean {
    // Assume a method to check current request user context or property flags
    return this.role === 'client' && this.verified;
  }

  private getLongName(): string {
    return `Mr. ${this.firstName} ${this.middleName} ${this.lastName}`;
  }

  private getShortName(): string {
    return `${this.firstName[0]}. ${this.lastName}`;
  }
}
```

---

## 3. Thin Controllers & Business Logic in Services

**Principle:**  
Controllers should be lean—handle HTTP requests/responses only. All business logic belongs in service classes.

**Bad Example (Controller with Business Logic):**

```typescript
// controllers/article.controller.ts (Bad)
import { Controller, Post, UploadedFile, UseInterceptors } from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';

@Controller('articles')
export class ArticleController {
  @Post()
  @UseInterceptors(FileInterceptor('image'))
  async create(@UploadedFile() image: Express.Multer.File) {
    if (image) {
      // Business logic for handling file upload inside controller
      // Not ideal!
      await image.mv('/path/to/temp');
    }
    // ... other logic
  }
}
```

**Good Example:**

```typescript
// controllers/article.controller.ts (Good)
import { Controller, Post, UploadedFile, UseInterceptors } from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { ArticleService } from './article.service';

@Controller('articles')
export class ArticleController {
  constructor(private readonly articleService: ArticleService) {}

  @Post()
  @UseInterceptors(FileInterceptor('image'))
  async create(@UploadedFile() image: Express.Multer.File) {
    await this.articleService.handleUploadedImage(image);
    // ... other HTTP concerns
  }
}
```

And in the service:

```typescript
// services/article.service.ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class ArticleService {
  async handleUploadedImage(image: Express.Multer.File): Promise<void> {
    if (image) {
      // Perform file system operation or delegate to a dedicated file service
      await image.mv('/path/to/temp');
    }
  }
}
```

---

## 4. Use DTOs, Validation, & Data Sanitization

**Principle:**  
Define explicit DTOs for input and output. Use Pipes with `class-validator` and `class-transformer` to enforce validation and sanitization.

**DTO Example:**

```typescript
// dtos/create-user.dto.ts
import { IsEmail, IsNotEmpty, MinLength } from 'class-validator';
import { Transform } from 'class-transformer';

export class CreateUserDto {
  @IsNotEmpty()
  @Transform(({ value }) => value.trim())
  username: string;

  @IsEmail()
  @Transform(({ value }) => value.toLowerCase().trim())
  email: string;

  @MinLength(6)
  password: string;
}
```

**Global Validation Pipe Setup (main.ts):**

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,        // Strip properties that have no decorators
      forbidNonWhitelisted: true,
      transform: true,        // Automatically convert payloads to DTO instances
    }),
  );
  await app.listen(3000);
}
bootstrap();
```

---

## 5. Avoid Repeating Yourself (DRY)

**Principle:**  
Abstract shared logic into helper functions, services, or custom pipes to avoid duplication.

**Bad Example:**

```typescript
// controllers/sample.controller.ts (Bad)
import { Controller, Get } from '@nestjs/common';
import { Repository } from 'typeorm';
import { User } from './entities/user.entity';

@Controller('sample')
export class SampleController {
  constructor(private readonly userRepository: Repository<User>) {}

  @Get('active')
  async getActiveUsers() {
    return this.userRepository.find({ where: { active: true, deletedAt: null } });
  }

  @Get('verified')
  async getVerifiedUsers() {
    return this.userRepository.find({ where: { verified: true, deletedAt: null } });
  }
}
```

**Good Example (Using Custom Repository or Service Methods):**

```typescript
// repositories/user.repository.ts (with a custom scope)
import { EntityRepository, Repository } from 'typeorm';
import { User } from '../entities/user.entity';

@EntityRepository(User)
export class UserRepository extends Repository<User> {
  getActive() {
    return this.find({ where: { active: true, deletedAt: null } });
  }

  getVerified() {
    return this.find({ where: { verified: true, deletedAt: null } });
  }
}
```

Then inject and use the repository in your service or controller.

---

## 6. Leverage TypeORM’s Features Over Raw SQL

**Principle:**  
Use the ORM’s expressive methods for querying, eager loading, and relationship management instead of raw SQL queries.

**Bad Example (Raw SQL):**

```typescript
// Using raw SQL queries (Bad)
await dataSource.query(`
  SELECT * FROM users
  WHERE id IN (
    SELECT user_id FROM posts WHERE published = true
  )
`);
```

**Good Example (Using TypeORM):**

```typescript
// Using TypeORM with relations and scopes (Good)
const users = await this.userRepository.find({
  relations: ['posts'],
  where: { posts: { published: true } },
});
```

---

Below is a dedicated section on how to avoid and protect against SQL injection in your NestJS application when using TypeORM. The focus is on leveraging safe practices such as parameterized queries, repository methods, and query builders rather than concatenating strings. These techniques ensure that your application never directly interpolates user input into SQL statements.

---

## 7 Protecting Against SQL Injection

### a. Use Parameterized Queries & Query Builders

When you need to execute custom SQL queries, never concatenate user input into the query string. Instead, use TypeORM’s query builder or parameterized queries that automatically escape inputs.

**Bad Example (Vulnerable to SQL Injection):**

```typescript
// Do not do this!
const userInput = "some user input";
await dataSource.query(`SELECT * FROM users WHERE username = '${userInput}'`);
```

**Good Example (Using Parameterized Queries):**

```typescript
// Using query builder for safety:
const userInput = "some user input";
const user = await dataSource
  .createQueryBuilder()
  .select('user')
  .from(User, 'user')
  .where('user.username = :username', { username: userInput })
  .getOne();
```

### b. Rely on Repository Methods

TypeORM’s repository methods (e.g., `find`, `findOne`, and `save`) automatically use parameter binding, so avoid raw SQL queries when possible.

**Example:**

```typescript
// Instead of writing raw SQL, use repository methods:
const user = await userRepository.findOne({ where: { username: userInput } });
```

### c. Avoid Dynamic SQL Construction

If you must build SQL dynamically, validate and sanitize inputs rigorously before incorporating them into queries. However, the preferred approach is to use parameters or query builders.

**Bad Example:**

```typescript
// Dynamic SQL creation using user input directly (bad)
const sortBy = req.query.sortBy; // e.g., "username"
const query = `SELECT * FROM users ORDER BY ${sortBy}`;
await dataSource.query(query);
```

**Good Example:**

```typescript
// Restrict dynamic parts to a whitelist and use parameters
const validSortFields = ['username', 'createdAt'];
const sortBy = validSortFields.includes(req.query.sortBy) ? req.query.sortBy : 'username';
const users = await dataSource
  .createQueryBuilder('user')
  .orderBy(`user.${sortBy}`, 'ASC')
  .getMany();
```

### d. Use ORM-Provided Methods for Data Manipulation

When working with inserts, updates, or deletes, always use methods provided by TypeORM (e.g., `create`, `save`, `update`, and `delete`). These methods safely handle the underlying SQL.

**Example:**

```typescript
// Creating a new user safely using repository methods
const newUser = userRepository.create({
  username: dto.username,
  email: dto.email,
  password: dto.password,
});
await userRepository.save(newUser);
```

---

## 8. Mass Assignment & Entity Creation

**Principle:**  
Use repository or ORM methods to create entities from DTOs rather than manually assigning properties.

**Bad Example:**

```typescript
// Manually assigning properties (Bad)
const user = new User();
user.username = dto.username;
user.email = dto.email;
user.password = dto.password;
await user.save();
```

**Good Example:**

```typescript
// Using repository create() for mass assignment (Good)
const user = this.userRepository.create(dto);
await this.userRepository.save(user);
```

---

## 9. Eager Loading & Avoiding N+1 Queries

**Principle:**  
Always load related data eagerly when needed to avoid multiple queries.

**Bad Example (N+1 problem):**

```typescript
// In a controller, iterating without eager loading (Bad)
const users = await this.userRepository.find();
for (const user of users) {
  console.log(user.profile.name); // Triggers extra query per user if profile is lazy-loaded
}
```

**Good Example:**

```typescript
// Eager loading the relation (Good)
const users = await this.userRepository.find({ relations: ['profile'] });
for (const user of users) {
  console.log(user.profile.name);
}
```

---

## 10. Chunking Data for Heavy Tasks

**Principle:**  
For operations on large datasets, process records in chunks to avoid memory issues.

**Bad Example:**

```typescript
// Loading all records at once (Bad)
const users = await this.userRepository.find();
users.forEach(user => {
  // Process user...
});
```

**Good Example:**

```typescript
// Using query builder with chunking (Good)
await this.userRepository
  .createQueryBuilder('user')
  .stream(async stream => {
    for await (const user of stream) {
      // Process each user record in a memory-efficient way
    }
  });
```

---

## 11. Use Dependency Injection (IoC) Instead of New Class

**Principle:**  
Always use NestJS’s DI container to inject dependencies. This makes testing and maintenance easier.

**Bad Example:**

```typescript
// Direct instantiation (Bad)
const service = new SomeService();
service.doWork();
```

**Good Example:**

```typescript
// Using constructor injection (Good)
import { Injectable } from '@nestjs/common';

@Injectable()
export class SomeConsumerService {
  constructor(private readonly someService: SomeService) {}

  doSomething() {
    this.someService.doWork();
  }
}
```

---

## 12. Avoid Direct Access to Environment Variables

**Principle:**  
Never use `process.env` directly in your application code. Instead, load these into configuration files and access them via the `ConfigService`.

**Bad Example:**

```typescript
const apiKey = process.env.API_KEY;
```

**Good Example:**

```typescript
// config/api.config.ts
export default () => ({
  apiKey: process.env.API_KEY,
});

// app.module.ts
import { ConfigModule } from '@nestjs/config';
import apiConfig from './config/api.config';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [apiConfig],
      isGlobal: true,
    }),
    // ...
  ],
})
export class AppModule {}

// Anywhere in your code:
import { ConfigService } from '@nestjs/config';

constructor(private configService: ConfigService) {
  const apiKey = this.configService.get<string>('apiKey');
}
```

---

## 13. Configuration & Constants Instead of Hard-Coded Text

**Principle:**  
Keep strings, dates, or keys in dedicated configuration or constants files instead of scattering them in your code.

**Bad Example:**

```typescript
if (user.role === 'admin') {
  // ...
}
```

**Good Example:**

```typescript
// entities/user.entity.ts
export enum UserRole {
  ADMIN = 'admin',
  USER = 'user',
}

// usage
if (user.role === UserRole.ADMIN) {
  // ...
}
```

---

## 14. Naming Conventions & File Organization

**Principle:**  
Follow NestJS (and broader TypeScript) naming conventions for files, classes, methods, and variables. For example:
- **Controllers:** Use singular names (e.g. `UserController`)
- **Services:** Use descriptive names (e.g. `AuthService`)
- **DTOs & Entities:** Use PascalCase for classes
- **Files & Folders:** Use kebab-case or camelCase consistently

**Good Example:**

```
src/
 ├── modules/
 │    ├── user/
 │    │    ├── controllers/user.controller.ts
 │    │    ├── dtos/create-user.dto.ts
 │    │    ├── entities/user.entity.ts
 │    │    ├── services/user.service.ts
 │    │    └── user.module.ts
```

---

## 15. Convention Over Configuration

**Principle:**  
Stick to NestJS defaults and conventions (like naming, module structure, and decorators) to minimize extra configuration and keep your codebase consistent.

**Example:**  
Define your module, controller, and service using built-in decorators without over-configuring unless necessary.

```typescript
// user.module.ts
import { Module } from '@nestjs/common';
import { UserController } from './controllers/user.controller';
import { UserService } from './services/user.service';

@Module({
  controllers: [UserController],
  providers: [UserService],
})
export class UserModule {}
```

---

## 16. Use Shorter, Readable Syntax Where Possible

**Principle:**  
Leverage TypeScript and NestJS shorthand to make your code cleaner. For example, use destructuring, concise arrow functions, and built-in helper functions.

**Examples:**

```typescript
// Instead of:
const username = request.body.username ? request.body.username.trim() : null;

// Use:
const { username = '' } = request.body;
const cleanedUsername = username.trim();
```

And in dependency injection, use parameter properties:

```typescript
constructor(private readonly userService: UserService) {}
```

---

## 17. Use Built-In NestJS Tools Over Third-Party Ones

**Principle:**  
Favor NestJS’s standard modules and widely adopted community packages rather than exotic third-party solutions. This ensures maintainability and easier onboarding for new developers.

**Example:**  
- Use `@nestjs/jwt` for JWT authentication.
- Use `@nestjs/config` for configuration management.
- Use `@nestjs/swagger` for API documentation.

---

## 18. Testing Best Practices

**Principle:**  
Write unit and end-to-end tests for your modules. Use NestJS’s testing utilities to inject dependencies and create isolated test environments.

**Example (Unit Test for a Service):**

```typescript
// services/user.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UserService } from './user.service';
import { Repository } from 'typeorm';
import { User } from '../entities/user.entity';

describe('UserService', () => {
  let service: UserService;
  let userRepository: Repository<User>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserService,
        {
          provide: 'UserRepositoryToken', // if using custom provider tokens
          useValue: {
            create: jest.fn(),
            save: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<UserService>(UserService);
    userRepository = module.get<Repository<User>>('UserRepositoryToken');
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  // Additional tests...
});
```

---

## 19. Database Best Practices with TypeORM

### a. Use Enums & Explicit Column Options

**Example:**

```typescript
// entities/user.entity.ts
import { Entity, PrimaryGeneratedColumn, Column } from 'typeorm';

export enum UserRole {
  ADMIN = 'admin',
  USER = 'user',
}

@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  username: string;

  @Column({ unique: true })
  email: string;

  @Column()
  password: string;

  @Column({ type: 'enum', enum: UserRole, default: UserRole.USER })
  role: UserRole;
}
```

### b. Use Transactions for Multi-Step Operations

**Example:**

```typescript
// services/user.service.ts
import { Injectable } from '@nestjs/common';
import { DataSource } from 'typeorm';
import { User } from '../entities/user.entity';
import { CreateUserDto } from '../dtos/create-user.dto';

@Injectable()
export class UserService {
  constructor(private readonly dataSource: DataSource) {}

  async createUser(dto: CreateUserDto): Promise<User> {
    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();
    try {
      const user = queryRunner.manager.create(User, dto);
      await queryRunner.manager.save(user);
      // Perform additional operations if necessary
      await queryRunner.commitTransaction();
      return user;
    } catch (error) {
      await queryRunner.rollbackTransaction();
      throw error;
    } finally {
      await queryRunner.release();
    }
  }
}
```

### c. Manage Schema Changes with Migrations

**Principle:**  
Keep your database schema under version control using TypeORM migrations.

**Example Migration File:**

```typescript
// migrations/1661234567890-CreateUserTable.ts
import { MigrationInterface, QueryRunner, Table } from 'typeorm';

export class CreateUserTable1661234567890 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.createTable(
      new Table({
        name: 'users',
        columns: [
          {
            name: 'id',
            type: 'uuid',
            isPrimary: true,
            generationStrategy: 'uuid',
            default: 'uuid_generate_v4()',
          },
          { name: 'username', type: 'varchar', isUnique: true },
          { name: 'email', type: 'varchar', isUnique: true },
          { name: 'password', type: 'varchar' },
          {
            name: 'role',
            type: 'enum',
            enum: ['admin', 'user'],
            default: "'user'",
          },
          { name: 'createdAt', type: 'timestamp', default: 'now()' },
          { name: 'updatedAt', type: 'timestamp', default: 'now()' },
        ],
      }),
      true,
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropTable('users');
  }
}
```

*Tip:* Use the TypeORM CLI to generate and run migrations.

---

## 20. Other Good Practices

- **Comment Wisely:** Use descriptive names for methods and variables to reduce the need for comments.
- **Avoid Inline HTML/JS in Server Code:** Keep front-end concerns separate from backend logic.
- **Use Standard NestJS Tools:** Stick to Nest’s ecosystem (e.g., Pipes, Guards, Interceptors) rather than mixing in non-standard solutions.
- **Leverage Language Files:** When internationalizing messages, use a dedicated translation module rather than hardcoding strings.
- **Do Not Override Framework Defaults:** Use NestJS’s built-in features unless you have a compelling reason not to.
- **Use Modern TypeScript Syntax:** Embrace async/await, decorators, and other modern patterns for clarity and conciseness.

---

Enjoy building your NestJS application with these guidelines in mind!
