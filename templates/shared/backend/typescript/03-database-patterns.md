# TypeScript データベースパターン

## Prisma ORM
```typescript
// schema.prisma
model User {
  id    Int     @id @default(autoincrement())
  name  String
  email String  @unique
  posts Post[]
}

model Post {
  id       Int    @id @default(autoincrement())
  title    String
  content  String
  authorId Int
  author   User   @relation(fields: [authorId], references: [id])
}

// TypeScript使用例
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

class UserRepository {
  async create(data: { name: string; email: string }) {
    return prisma.user.create({ data });
  }

  async findById(id: number) {
    return prisma.user.findUnique({
      where: { id },
      include: { posts: true }
    });
  }

  async update(id: number, data: Partial<User>) {
    return prisma.user.update({
      where: { id },
      data
    });
  }
}
```

## TypeORM
```typescript
import { Entity, PrimaryGeneratedColumn, Column, OneToMany } from 'typeorm';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ length: 100 })
  name: string;

  @Column({ unique: true })
  email: string;

  @OneToMany(() => Post, post => post.author)
  posts: Post[];
}

// Repository使用
import { Repository } from 'typeorm';

class UserService {
  constructor(private userRepo: Repository<User>) {}

  async createUser(userData: CreateUserDto): Promise<User> {
    const user = this.userRepo.create(userData);
    return this.userRepo.save(user);
  }
}
```