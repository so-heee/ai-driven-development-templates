# データベースパターンガイド

## データベース設計原則

### リレーショナル設計の基本
```sql
-- 正規化された設計例
-- ユーザーテーブル
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,
    role user_role DEFAULT 'user',
    email_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- プロフィールテーブル（1対1関係）
CREATE TABLE user_profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    bio TEXT,
    avatar_url VARCHAR(500),
    phone_number VARCHAR(20),
    date_of_birth DATE,
    timezone VARCHAR(50) DEFAULT 'UTC',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 投稿テーブル（1対多関係）
CREATE TABLE posts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    author_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(200) NOT NULL,
    content TEXT NOT NULL,
    status post_status DEFAULT 'draft',
    published_at TIMESTAMP,
    view_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- タグテーブル（多対多関係）
CREATE TABLE tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 投稿-タグ中間テーブル
CREATE TABLE post_tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    tag_id UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(post_id, tag_id)
);

-- インデックス設計
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_posts_author_id ON posts(author_id);
CREATE INDEX idx_posts_status ON posts(status);
CREATE INDEX idx_posts_published_at ON posts(published_at) WHERE status = 'published';
CREATE INDEX idx_post_tags_post_id ON post_tags(post_id);
CREATE INDEX idx_post_tags_tag_id ON post_tags(tag_id);

-- 複合インデックス
CREATE INDEX idx_posts_author_status ON posts(author_id, status);
CREATE INDEX idx_posts_status_created ON posts(status, created_at DESC);
```

### カスタム型とENUM
```sql
-- カスタム型定義
CREATE TYPE user_role AS ENUM ('user', 'moderator', 'admin');
CREATE TYPE post_status AS ENUM ('draft', 'published', 'archived');
CREATE TYPE notification_type AS ENUM ('comment', 'like', 'follow', 'system');

-- 住所用の複合型
CREATE TYPE address AS (
    street VARCHAR(100),
    city VARCHAR(50),
    state VARCHAR(50),
    country VARCHAR(50),
    postal_code VARCHAR(20)
);

-- JSON型の活用
CREATE TABLE user_settings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    preferences JSONB NOT NULL DEFAULT '{}',
    theme_config JSONB,
    notification_settings JSONB NOT NULL DEFAULT '{"email": true, "push": true}',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- JSONBインデックス
CREATE INDEX idx_user_settings_preferences ON user_settings USING GIN (preferences);
CREATE INDEX idx_notification_email ON user_settings USING GIN ((notification_settings->'email'));
```

## ORM/ODM パターン

### Prisma パターン
```typescript
// schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id            String    @id @default(cuid())
  email         String    @unique
  passwordHash  String    @map("password_hash")
  name          String
  role          Role      @default(USER)
  emailVerified Boolean   @default(false) @map("email_verified")
  createdAt     DateTime  @default(now()) @map("created_at")
  updatedAt     DateTime  @updatedAt @map("updated_at")
  
  // リレーション
  profile       UserProfile?
  posts         Post[]
  comments      Comment[]
  sessions      Session[]
  
  @@map("users")
}

model UserProfile {
  id          String    @id @default(cuid())
  userId      String    @unique @map("user_id")
  bio         String?
  avatarUrl   String?   @map("avatar_url")
  phoneNumber String?   @map("phone_number")
  dateOfBirth DateTime? @map("date_of_birth")
  timezone    String    @default("UTC")
  createdAt   DateTime  @default(now()) @map("created_at")
  updatedAt   DateTime  @updatedAt @map("updated_at")
  
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@map("user_profiles")
}

model Post {
  id          String      @id @default(cuid())
  authorId    String      @map("author_id")
  title       String
  content     String
  status      PostStatus  @default(DRAFT)
  publishedAt DateTime?   @map("published_at")
  viewCount   Int         @default(0) @map("view_count")
  createdAt   DateTime    @default(now()) @map("created_at")
  updatedAt   DateTime    @updatedAt @map("updated_at")
  
  // リレーション
  author   User       @relation(fields: [authorId], references: [id], onDelete: Cascade)
  comments Comment[]
  tags     PostTag[]
  
  @@index([authorId])
  @@index([status])
  @@index([publishedAt])
  @@map("posts")
}

enum Role {
  USER
  MODERATOR
  ADMIN
}

enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}

// 使用例 - リポジトリパターン
class UserRepository {
  constructor(private prisma: PrismaClient) {}
  
  async findById(id: string): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: { id },
      include: {
        profile: true,
        posts: {
          where: { status: 'PUBLISHED' },
          orderBy: { createdAt: 'desc' },
          take: 10
        }
      }
    });
  }
  
  async findByEmail(email: string): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: { email },
      include: { profile: true }
    });
  }
  
  async create(data: CreateUserData): Promise<User> {
    return this.prisma.user.create({
      data: {
        email: data.email,
        passwordHash: data.passwordHash,
        name: data.name,
        role: data.role || 'USER',
        profile: data.profile ? {
          create: data.profile
        } : undefined
      },
      include: { profile: true }
    });
  }
  
  async update(id: string, data: UpdateUserData): Promise<User> {
    return this.prisma.user.update({
      where: { id },
      data: {
        ...data,
        updatedAt: new Date(),
        profile: data.profile ? {
          upsert: {
            create: data.profile,
            update: data.profile
          }
        } : undefined
      },
      include: { profile: true }
    });
  }
  
  async findWithPagination(params: {
    page: number;
    limit: number;
    search?: string;
    role?: Role;
  }): Promise<{ users: User[]; total: number }> {
    const { page, limit, search, role } = params;
    const skip = (page - 1) * limit;
    
    const where = {
      ...(search && {
        OR: [
          { name: { contains: search, mode: 'insensitive' as const } },
          { email: { contains: search, mode: 'insensitive' as const } }
        ]
      }),
      ...(role && { role })
    };
    
    const [users, total] = await Promise.all([
      this.prisma.user.findMany({
        where,
        skip,
        take: limit,
        orderBy: { createdAt: 'desc' },
        include: { profile: true }
      }),
      this.prisma.user.count({ where })
    ]);
    
    return { users, total };
  }
  
  async delete(id: string): Promise<void> {
    await this.prisma.user.delete({
      where: { id }
    });
  }
}
```

### MongoDB (Mongoose) パターン
```typescript
import mongoose, { Schema, Document } from 'mongoose';

// スキーマ定義
interface IUser extends Document {
  email: string;
  passwordHash: string;
  name: string;
  role: 'user' | 'moderator' | 'admin';
  profile: {
    bio?: string;
    avatarUrl?: string;
    phoneNumber?: string;
    dateOfBirth?: Date;
    timezone: string;
  };
  preferences: {
    theme: string;
    notifications: {
      email: boolean;
      push: boolean;
      sms: boolean;
    };
    privacy: {
      profileVisible: boolean;
      showEmail: boolean;
    };
  };
  createdAt: Date;
  updatedAt: Date;
}

const userSchema = new Schema<IUser>({
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true,
    validate: {
      validator: (email: string) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email),
      message: '有効なメールアドレスを入力してください'
    }
  },
  passwordHash: {
    type: String,
    required: true,
    minlength: 8
  },
  name: {
    type: String,
    required: true,
    trim: true,
    maxlength: 100
  },
  role: {
    type: String,
    enum: ['user', 'moderator', 'admin'],
    default: 'user'
  },
  profile: {
    bio: { type: String, maxlength: 500 },
    avatarUrl: String,
    phoneNumber: String,
    dateOfBirth: Date,
    timezone: { type: String, default: 'UTC' }
  },
  preferences: {
    theme: { type: String, default: 'light' },
    notifications: {
      email: { type: Boolean, default: true },
      push: { type: Boolean, default: true },
      sms: { type: Boolean, default: false }
    },
    privacy: {
      profileVisible: { type: Boolean, default: true },
      showEmail: { type: Boolean, default: false }
    }
  }
}, {
  timestamps: true,
  toJSON: { virtuals: true },
  toObject: { virtuals: true }
});

// インデックス
userSchema.index({ email: 1 });
userSchema.index({ role: 1 });
userSchema.index({ 'profile.phoneNumber': 1 });
userSchema.index({ createdAt: -1 });

// 仮想フィールド
userSchema.virtual('posts', {
  ref: 'Post',
  localField: '_id',
  foreignField: 'author'
});

// インスタンスメソッド
userSchema.methods.toJSON = function() {
  const user = this.toObject();
  delete user.passwordHash;
  return user;
};

userSchema.methods.comparePassword = async function(password: string): Promise<boolean> {
  return bcrypt.compare(password, this.passwordHash);
};

// スタティックメソッド
userSchema.statics.findByEmail = function(email: string) {
  return this.findOne({ email: email.toLowerCase() });
};

userSchema.statics.findActiveUsers = function() {
  return this.find({ 
    role: { $in: ['user', 'moderator', 'admin'] },
    'preferences.privacy.profileVisible': true 
  });
};

// ミドルウェア
userSchema.pre('save', async function(next) {
  if (!this.isModified('passwordHash')) return next();
  
  const salt = await bcrypt.genSalt(10);
  this.passwordHash = await bcrypt.hash(this.passwordHash, salt);
  next();
});

userSchema.pre('findOneAndUpdate', function() {
  this.set({ updatedAt: new Date() });
});

const User = mongoose.model<IUser>('User', userSchema);

// リポジトリパターンの実装
class MongoUserRepository {
  async findById(id: string): Promise<IUser | null> {
    return User.findById(id).populate('posts').exec();
  }
  
  async findByEmail(email: string): Promise<IUser | null> {
    return User.findByEmail(email);
  }
  
  async create(userData: Partial<IUser>): Promise<IUser> {
    const user = new User(userData);
    return user.save();
  }
  
  async update(id: string, data: Partial<IUser>): Promise<IUser | null> {
    return User.findByIdAndUpdate(
      id,
      { $set: data },
      { new: true, runValidators: true }
    );
  }
  
  async findWithFilters(filters: {
    page: number;
    limit: number;
    search?: string;
    role?: string;
  }): Promise<{ users: IUser[]; total: number }> {
    const { page, limit, search, role } = filters;
    const skip = (page - 1) * limit;
    
    const query: any = {};
    
    if (search) {
      query.$or = [
        { name: { $regex: search, $options: 'i' } },
        { email: { $regex: search, $options: 'i' } }
      ];
    }
    
    if (role) {
      query.role = role;
    }
    
    const [users, total] = await Promise.all([
      User.find(query)
        .skip(skip)
        .limit(limit)
        .sort({ createdAt: -1 })
        .exec(),
      User.countDocuments(query)
    ]);
    
    return { users, total };
  }
  
  async aggregateUserStats(): Promise<any[]> {
    return User.aggregate([
      {
        $group: {
          _id: '$role',
          count: { $sum: 1 },
          avgAge: {
            $avg: {
              $divide: [
                { $subtract: [new Date(), '$profile.dateOfBirth'] },
                365 * 24 * 60 * 60 * 1000
              ]
            }
          }
        }
      },
      {
        $sort: { count: -1 }
      }
    ]);
  }
}
```

## トランザクション管理

### ACID トランザクションパターン
```typescript
// Prisma でのトランザクション
class TransactionalService {
  constructor(private prisma: PrismaClient) {}
  
  // インタラクティブトランザクション
  async transferUserData(fromUserId: string, toUserId: string) {
    return this.prisma.$transaction(async (tx) => {
      // ユーザー存在確認
      const fromUser = await tx.user.findUnique({
        where: { id: fromUserId }
      });
      
      const toUser = await tx.user.findUnique({
        where: { id: toUserId }
      });
      
      if (!fromUser || !toUser) {
        throw new Error('ユーザーが見つかりません');
      }
      
      // 投稿の所有者を変更
      await tx.post.updateMany({
        where: { authorId: fromUserId },
        data: { authorId: toUserId }
      });
      
      // コメントの所有者を変更
      await tx.comment.updateMany({
        where: { authorId: fromUserId },
        data: { authorId: toUserId }
      });
      
      // 元のユーザーを削除
      await tx.user.delete({
        where: { id: fromUserId }
      });
      
      return toUser;
    });
  }
  
  // バッチトランザクション
  async createUserWithPosts(userData: CreateUserData, postsData: CreatePostData[]) {
    const operations = [
      this.prisma.user.create({
        data: userData
      }),
      ...postsData.map(postData =>
        this.prisma.post.create({
          data: {
            ...postData,
            authorId: userData.id
          }
        })
      )
    ];
    
    return this.prisma.$transaction(operations);
  }
  
  // 分散トランザクション（Saga パターン）
  async processOrderSaga(orderData: OrderData) {
    const sagaSteps = [
      {
        action: () => this.createOrder(orderData),
        compensation: (orderId: string) => this.cancelOrder(orderId)
      },
      {
        action: (orderId: string) => this.processPayment(orderData.paymentInfo),
        compensation: (paymentId: string) => this.refundPayment(paymentId)
      },
      {
        action: (paymentId: string) => this.reserveInventory(orderData.items),
        compensation: (reservationId: string) => this.releaseInventory(reservationId)
      },
      {
        action: (reservationId: string) => this.scheduleShipping(orderData.shippingInfo),
        compensation: (shippingId: string) => this.cancelShipping(shippingId)
      }
    ];
    
    const executedSteps: any[] = [];
    
    try {
      let previousResult = null;
      
      for (const step of sagaSteps) {
        const result = await step.action(previousResult);
        executedSteps.push({ result, compensation: step.compensation });
        previousResult = result;
      }
      
      return previousResult;
    } catch (error) {
      // 実行された操作を逆順で補償
      for (const step of executedSteps.reverse()) {
        try {
          await step.compensation(step.result);
        } catch (compensationError) {
          console.error('Compensation failed:', compensationError);
        }
      }
      
      throw error;
    }
  }
}

// MongoDB でのトランザクション
class MongoTransactionalService {
  constructor(private mongoose: typeof mongoose) {}
  
  async transferDocuments(fromUserId: string, toUserId: string) {
    const session = await this.mongoose.startSession();
    
    try {
      await session.withTransaction(async () => {
        // 操作1: ユーザー確認
        const fromUser = await User.findById(fromUserId).session(session);
        const toUser = await User.findById(toUserId).session(session);
        
        if (!fromUser || !toUser) {
          throw new Error('ユーザーが見つかりません');
        }
        
        // 操作2: 投稿の移管
        await Post.updateMany(
          { author: fromUserId },
          { author: toUserId },
          { session }
        );
        
        // 操作3: ユーザー削除
        await User.findByIdAndDelete(fromUserId, { session });
      });
    } finally {
      await session.endSession();
    }
  }
  
  // 楽観的ロッキング
  async updateWithOptimisticLocking(id: string, data: any, version: number) {
    const session = await this.mongoose.startSession();
    
    try {
      await session.withTransaction(async () => {
        const doc = await User.findById(id).session(session);
        
        if (!doc) {
          throw new Error('ドキュメントが見つかりません');
        }
        
        if (doc.__v !== version) {
          throw new Error('ドキュメントが他のプロセスによって更新されています');
        }
        
        await User.findByIdAndUpdate(
          id,
          { ...data, $inc: { __v: 1 } },
          { session }
        );
      });
    } finally {
      await session.endSession();
    }
  }
}
```

## クエリ最適化

### N+1問題の解決
```typescript
// Prisma でのincludeとselect最適化
class OptimizedQueryService {
  // ❌ N+1問題のある例
  async getUsersWithPostsBad(): Promise<any[]> {
    const users = await this.prisma.user.findMany();
    
    const usersWithPosts = await Promise.all(
      users.map(async (user) => ({
        ...user,
        posts: await this.prisma.post.findMany({
          where: { authorId: user.id }
        })
      }))
    );
    
    return usersWithPosts;
  }
  
  // ✅ 最適化された例
  async getUsersWithPostsOptimized(): Promise<any[]> {
    return this.prisma.user.findMany({
      include: {
        posts: {
          where: { status: 'PUBLISHED' },
          orderBy: { createdAt: 'desc' },
          take: 5
        },
        profile: true
      }
    });
  }
  
  // 大量データの効率的な処理
  async processLargeDataset(): Promise<void> {
    const batchSize = 1000;
    let cursor: string | undefined;
    
    do {
      const batch = await this.prisma.user.findMany({
        take: batchSize,
        ...(cursor && {
          cursor: { id: cursor },
          skip: 1
        }),
        orderBy: { id: 'asc' }
      });
      
      if (batch.length === 0) break;
      
      // バッチ処理
      await this.processBatch(batch);
      
      cursor = batch[batch.length - 1].id;
    } while (true);
  }
  
  // 集計クエリの最適化
  async getAnalytics() {
    const result = await this.prisma.$queryRaw`
      SELECT 
        u.role,
        COUNT(*)::int as user_count,
        AVG(EXTRACT(year FROM age(u.created_at)))::float as avg_account_age,
        COUNT(p.id)::int as total_posts
      FROM users u
      LEFT JOIN posts p ON u.id = p.author_id
      WHERE u.created_at >= NOW() - INTERVAL '1 year'
      GROUP BY u.role
      ORDER BY user_count DESC
    `;
    
    return result;
  }
}

// MongoDB でのクエリ最適化
class MongoOptimizedService {
  // 効率的な検索クエリ
  async searchUsersOptimized(searchTerm: string, page: number, limit: number) {
    const skip = (page - 1) * limit;
    
    // テキストインデックスを使用した検索
    const pipeline = [
      {
        $match: {
          $text: { $search: searchTerm }
        }
      },
      {
        $lookup: {
          from: 'posts',
          localField: '_id',
          foreignField: 'author',
          as: 'recentPosts',
          pipeline: [
            { $match: { status: 'published' } },
            { $sort: { createdAt: -1 } },
            { $limit: 3 }
          ]
        }
      },
      {
        $addFields: {
          score: { $meta: 'textScore' }
        }
      },
      {
        $sort: { score: { $meta: 'textScore' } }
      },
      {
        $skip: skip
      },
      {
        $limit: limit
      }
    ];
    
    return User.aggregate(pipeline);
  }
  
  // 大量データの効率的な更新
  async bulkUpdateUsers(updates: Array<{ id: string; data: any }>) {
    const bulkOps = updates.map(update => ({
      updateOne: {
        filter: { _id: update.id },
        update: { $set: update.data },
        upsert: false
      }
    }));
    
    return User.bulkWrite(bulkOps, { ordered: false });
  }
  
  // ストリーミング処理
  async processUsersStream(processFn: (user: any) => Promise<void>) {
    const cursor = User.find({}).cursor();
    
    for (let user = await cursor.next(); user != null; user = await cursor.next()) {
      await processFn(user);
    }
  }
}
```

## マイグレーション戦略

### スキーマ進化パターン
```typescript
// Prisma マイグレーション例
// migration_001_initial.sql
/*
  Warnings:
  - Initial migration
*/

-- CreateEnum
CREATE TYPE "Role" AS ENUM ('USER', 'MODERATOR', 'ADMIN');

-- CreateTable
CREATE TABLE "users" (
    "id" TEXT NOT NULL,
    "email" TEXT NOT NULL,
    "password_hash" TEXT NOT NULL,
    "name" TEXT NOT NULL,
    "role" "Role" NOT NULL DEFAULT 'USER',
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_at" TIMESTAMP(3) NOT NULL,

    CONSTRAINT "users_pkey" PRIMARY KEY ("id")
);

-- CreateIndex
CREATE UNIQUE INDEX "users_email_key" ON "users"("email");

// migration_002_add_profile.sql
-- CreateTable
CREATE TABLE "user_profiles" (
    "id" TEXT NOT NULL,
    "user_id" TEXT NOT NULL,
    "bio" TEXT,
    "avatar_url" TEXT,
    "phone_number" TEXT,
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_at" TIMESTAMP(3) NOT NULL,

    CONSTRAINT "user_profiles_pkey" PRIMARY KEY ("id")
);

-- CreateIndex
CREATE UNIQUE INDEX "user_profiles_user_id_key" ON "user_profiles"("user_id");

-- AddForeignKey
ALTER TABLE "user_profiles" ADD CONSTRAINT "user_profiles_user_id_fkey" 
FOREIGN KEY ("user_id") REFERENCES "users"("id") ON DELETE CASCADE ON UPDATE CASCADE;

// migration_003_add_posts.sql
-- CreateEnum
CREATE TYPE "PostStatus" AS ENUM ('DRAFT', 'PUBLISHED', 'ARCHIVED');

-- CreateTable
CREATE TABLE "posts" (
    "id" TEXT NOT NULL,
    "author_id" TEXT NOT NULL,
    "title" TEXT NOT NULL,
    "content" TEXT NOT NULL,
    "status" "PostStatus" NOT NULL DEFAULT 'DRAFT',
    "published_at" TIMESTAMP(3),
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_at" TIMESTAMP(3) NOT NULL,

    CONSTRAINT "posts_pkey" PRIMARY KEY ("id")
);

-- CreateIndex
CREATE INDEX "posts_author_id_idx" ON "posts"("author_id");
CREATE INDEX "posts_status_idx" ON "posts"("status");

-- 安全なマイグレーション（カラム追加）
-- migration_004_add_view_count.sql
ALTER TABLE "posts" ADD COLUMN "view_count" INTEGER NOT NULL DEFAULT 0;

-- インデックス追加（オンライン）
CREATE INDEX CONCURRENTLY "idx_posts_published_at" ON "posts"("published_at") 
WHERE "status" = 'PUBLISHED';

// データマイグレーション
// migration_005_migrate_legacy_data.sql
-- レガシーデータの移行
INSERT INTO "users" (id, email, password_hash, name, role, created_at, updated_at)
SELECT 
  gen_random_uuid(),
  legacy_email,
  legacy_password,
  legacy_name,
  CASE 
    WHEN legacy_is_admin THEN 'ADMIN'::Role
    ELSE 'USER'::Role
  END,
  legacy_created_at,
  NOW()
FROM legacy_users
WHERE NOT EXISTS (
  SELECT 1 FROM users WHERE email = legacy_users.legacy_email
);

// ロールバック可能なマイグレーション
class ReversibleMigration {
  async up(prisma: PrismaClient) {
    // 新しいカラムを追加
    await prisma.$executeRaw`
      ALTER TABLE "users" ADD COLUMN "last_login_at" TIMESTAMP;
    `;
    
    // デフォルト値を設定
    await prisma.$executeRaw`
      UPDATE "users" SET "last_login_at" = "created_at";
    `;
  }
  
  async down(prisma: PrismaClient) {
    // カラムを削除
    await prisma.$executeRaw`
      ALTER TABLE "users" DROP COLUMN "last_login_at";
    `;
  }
}

// ゼロダウンタイムマイグレーション戦略
class ZeroDowntimeMigration {
  // フェーズ1: 新カラムを追加（NULL許可）
  async phase1AddColumn(prisma: PrismaClient) {
    await prisma.$executeRaw`
      ALTER TABLE "users" ADD COLUMN "new_status" TEXT;
    `;
  }
  
  // フェーズ2: アプリケーションで新旧両方のカラムを更新
  async phase2UpdateApplication() {
    // アプリケーションコードの更新
    // 読み取り: 新カラム優先、フォールバック
    // 書き込み: 新旧両方に書き込み
  }
  
  // フェーズ3: 既存データを新カラムに移行
  async phase3MigrateData(prisma: PrismaClient) {
    await prisma.$executeRaw`
      UPDATE "users" 
      SET "new_status" = "old_status" 
      WHERE "new_status" IS NULL;
    `;
  }
  
  // フェーズ4: 新カラムにNOT NULL制約を追加
  async phase4AddConstraint(prisma: PrismaClient) {
    await prisma.$executeRaw`
      ALTER TABLE "users" ALTER COLUMN "new_status" SET NOT NULL;
    `;
  }
  
  // フェーズ5: 旧カラムを削除
  async phase5DropOldColumn(prisma: PrismaClient) {
    await prisma.$executeRaw`
      ALTER TABLE "users" DROP COLUMN "old_status";
    `;
    
    await prisma.$executeRaw`
      ALTER TABLE "users" RENAME COLUMN "new_status" TO "status";
    `;
  }
}
```