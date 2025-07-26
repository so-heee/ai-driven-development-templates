# APIパターンガイド

## RESTful API設計

### リソース設計の基本原則
```typescript
// 適切なURL設計
// ✅ 良い例
GET    /api/v1/users              // ユーザー一覧取得
POST   /api/v1/users              // ユーザー作成
GET    /api/v1/users/:id          // 特定ユーザー取得
PUT    /api/v1/users/:id          // ユーザー全体更新
PATCH  /api/v1/users/:id          // ユーザー部分更新
DELETE /api/v1/users/:id          // ユーザー削除

// ネストしたリソース
GET    /api/v1/users/:id/posts    // ユーザーの投稿一覧
POST   /api/v1/users/:id/posts    // ユーザーの投稿作成
GET    /api/v1/posts/:id/comments // 投稿のコメント一覧

// ❌ 悪い例
GET    /api/getUsers
POST   /api/createUser
GET    /api/user/get/:id
DELETE /api/deleteUserById/:id
```

### HTTPステータスコードの適切な使用
```typescript
// Express.js での実装例
import { Request, Response } from 'express';

class UserController {
  // 成功レスポンス
  async getUsers(req: Request, res: Response) {
    try {
      const users = await userService.findAll();
      return res.status(200).json({
        success: true,
        data: users,
        message: 'ユーザー一覧を取得しました'
      });
    } catch (error) {
      return res.status(500).json({
        success: false,
        error: 'サーバーエラーが発生しました'
      });
    }
  }

  async createUser(req: Request, res: Response) {
    try {
      const userData = req.body;
      
      // バリデーション
      const validation = validateUserData(userData);
      if (!validation.isValid) {
        return res.status(400).json({
          success: false,
          error: 'バリデーションエラー',
          details: validation.errors
        });
      }
      
      // 重複チェック
      const existingUser = await userService.findByEmail(userData.email);
      if (existingUser) {
        return res.status(409).json({
          success: false,
          error: 'このメールアドレスは既に使用されています'
        });
      }
      
      const newUser = await userService.create(userData);
      return res.status(201).json({
        success: true,
        data: newUser,
        message: 'ユーザーが作成されました'
      });
    } catch (error) {
      return res.status(500).json({
        success: false,
        error: 'ユーザー作成に失敗しました'
      });
    }
  }

  async getUserById(req: Request, res: Response) {
    try {
      const { id } = req.params;
      const user = await userService.findById(id);
      
      if (!user) {
        return res.status(404).json({
          success: false,
          error: 'ユーザーが見つかりません'
        });
      }
      
      return res.status(200).json({
        success: true,
        data: user
      });
    } catch (error) {
      return res.status(500).json({
        success: false,
        error: 'サーバーエラーが発生しました'
      });
    }
  }

  async updateUser(req: Request, res: Response) {
    try {
      const { id } = req.params;
      const updateData = req.body;
      
      const user = await userService.findById(id);
      if (!user) {
        return res.status(404).json({
          success: false,
          error: 'ユーザーが見つかりません'
        });
      }
      
      const updatedUser = await userService.update(id, updateData);
      return res.status(200).json({
        success: true,
        data: updatedUser,
        message: 'ユーザー情報が更新されました'
      });
    } catch (error) {
      return res.status(500).json({
        success: false,
        error: 'ユーザー更新に失敗しました'
      });
    }
  }

  async deleteUser(req: Request, res: Response) {
    try {
      const { id } = req.params;
      
      const user = await userService.findById(id);
      if (!user) {
        return res.status(404).json({
          success: false,
          error: 'ユーザーが見つかりません'
        });
      }
      
      await userService.delete(id);
      return res.status(204).send(); // No Content
    } catch (error) {
      return res.status(500).json({
        success: false,
        error: 'ユーザー削除に失敗しました'
      });
    }
  }
}
```

### ページネーション・フィルタリング・ソート
```typescript
interface QueryParams {
  page?: number;
  limit?: number;
  sort?: string;
  order?: 'asc' | 'desc';
  search?: string;
  filter?: Record<string, any>;
}

class APIQueryBuilder {
  static buildQuery(params: QueryParams) {
    const {
      page = 1,
      limit = 10,
      sort = 'createdAt',
      order = 'desc',
      search,
      filter = {}
    } = params;

    // ページネーション
    const offset = (page - 1) * limit;

    // ソート条件
    const orderBy = { [sort]: order };

    // 検索条件
    const where: any = { ...filter };
    if (search) {
      where.OR = [
        { name: { contains: search, mode: 'insensitive' } },
        { email: { contains: search, mode: 'insensitive' } }
      ];
    }

    return {
      where,
      orderBy,
      skip: offset,
      take: limit
    };
  }

  static formatResponse<T>(
    data: T[],
    total: number,
    page: number,
    limit: number
  ) {
    const totalPages = Math.ceil(total / limit);
    
    return {
      data,
      pagination: {
        current: page,
        total: totalPages,
        count: data.length,
        totalCount: total,
        hasNext: page < totalPages,
        hasPrev: page > 1
      }
    };
  }
}

// 使用例
const getUsersWithPagination = async (req: Request, res: Response) => {
  try {
    const queryParams: QueryParams = {
      page: parseInt(req.query.page as string) || 1,
      limit: Math.min(parseInt(req.query.limit as string) || 10, 100),
      sort: req.query.sort as string || 'createdAt',
      order: (req.query.order as 'asc' | 'desc') || 'desc',
      search: req.query.search as string,
      filter: {
        ...(req.query.role && { role: req.query.role }),
        ...(req.query.status && { status: req.query.status })
      }
    };

    const query = APIQueryBuilder.buildQuery(queryParams);
    
    const [users, total] = await Promise.all([
      prisma.user.findMany(query),
      prisma.user.count({ where: query.where })
    ]);

    const response = APIQueryBuilder.formatResponse(
      users,
      total,
      queryParams.page!,
      queryParams.limit!
    );

    return res.status(200).json({
      success: true,
      ...response
    });
  } catch (error) {
    return res.status(500).json({
      success: false,
      error: 'ユーザー取得に失敗しました'
    });
  }
};
```

## API バリデーション

### リクエストバリデーション
```typescript
import Joi from 'joi';
import { Request, Response, NextFunction } from 'express';

// バリデーションスキーマ定義
const userSchemas = {
  create: Joi.object({
    name: Joi.string().min(1).max(100).required(),
    email: Joi.string().email().required(),
    password: Joi.string().min(8).pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/).required(),
    role: Joi.string().valid('user', 'admin').default('user'),
    profile: Joi.object({
      bio: Joi.string().max(500),
      avatar: Joi.string().uri(),
      phoneNumber: Joi.string().pattern(/^[0-9-+().\s]+$/)
    }).optional()
  }),
  
  update: Joi.object({
    name: Joi.string().min(1).max(100),
    email: Joi.string().email(),
    role: Joi.string().valid('user', 'admin'),
    profile: Joi.object({
      bio: Joi.string().max(500),
      avatar: Joi.string().uri(),
      phoneNumber: Joi.string().pattern(/^[0-9-+().\s]+$/)
    })
  }).min(1), // 最低1つのフィールドが必要
  
  query: Joi.object({
    page: Joi.number().integer().min(1).default(1),
    limit: Joi.number().integer().min(1).max(100).default(10),
    sort: Joi.string().valid('name', 'email', 'createdAt', 'updatedAt').default('createdAt'),
    order: Joi.string().valid('asc', 'desc').default('desc'),
    search: Joi.string().max(100),
    role: Joi.string().valid('user', 'admin'),
    status: Joi.string().valid('active', 'inactive')
  })
};

// バリデーションミドルウェア
const validate = (schema: Joi.Schema, property: 'body' | 'query' | 'params' = 'body') => {
  return (req: Request, res: Response, next: NextFunction) => {
    const { error, value } = schema.validate(req[property], {
      abortEarly: false,
      allowUnknown: false,
      stripUnknown: true
    });

    if (error) {
      const errors = error.details.map(detail => ({
        field: detail.path.join('.'),
        message: detail.message,
        value: detail.context?.value
      }));

      return res.status(400).json({
        success: false,
        error: 'バリデーションエラー',
        details: errors
      });
    }

    req[property] = value;
    next();
  };
};

// カスタムバリデーター
const customValidators = {
  uniqueEmail: async (email: string, userId?: string) => {
    const existingUser = await prisma.user.findUnique({
      where: { email }
    });
    
    if (existingUser && existingUser.id !== userId) {
      throw new Error('このメールアドレスは既に使用されています');
    }
  },
  
  strongPassword: (password: string) => {
    const minLength = 8;
    const hasUppercase = /[A-Z]/.test(password);
    const hasLowercase = /[a-z]/.test(password);
    const hasNumber = /\d/.test(password);
    const hasSpecialChar = /[!@#$%^&*(),.?":{}|<>]/.test(password);
    
    if (password.length < minLength) {
      throw new Error(`パスワードは${minLength}文字以上である必要があります`);
    }
    
    if (!hasUppercase || !hasLowercase || !hasNumber || !hasSpecialChar) {
      throw new Error('パスワードは大文字、小文字、数字、特殊文字を含む必要があります');
    }
  }
};

// 使用例
router.post('/users',
  validate(userSchemas.create),
  async (req: Request, res: Response) => {
    try {
      // 追加のカスタムバリデーション
      await customValidators.uniqueEmail(req.body.email);
      customValidators.strongPassword(req.body.password);
      
      const user = await userService.create(req.body);
      return res.status(201).json({
        success: true,
        data: user
      });
    } catch (error) {
      return res.status(400).json({
        success: false,
        error: error.message
      });
    }
  }
);
```

## ミドルウェアパターン

### 認証・認可ミドルウェア
```typescript
import jwt from 'jsonwebtoken';
import { Request, Response, NextFunction } from 'express';

interface AuthenticatedRequest extends Request {
  user?: {
    id: string;
    email: string;
    role: string;
  };
}

// JWT認証ミドルウェア
const authenticateToken = (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
  const authHeader = req.headers.authorization;
  const token = authHeader && authHeader.split(' ')[1];

  if (!token) {
    return res.status(401).json({
      success: false,
      error: 'アクセストークンが必要です'
    });
  }

  jwt.verify(token, process.env.JWT_SECRET!, (err, decoded) => {
    if (err) {
      return res.status(403).json({
        success: false,
        error: '無効なトークンです'
      });
    }

    req.user = decoded as any;
    next();
  });
};

// ロールベース認可ミドルウェア
const requireRole = (...roles: string[]) => {
  return (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({
        success: false,
        error: '認証が必要です'
      });
    }

    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        success: false,
        error: '権限が不足しています'
      });
    }

    next();
  };
};

// リソース所有者チェック
const requireOwnership = (resourceParam: string = 'id') => {
  return async (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    try {
      const resourceId = req.params[resourceParam];
      const userId = req.user?.id;

      if (!userId) {
        return res.status(401).json({
          success: false,
          error: '認証が必要です'
        });
      }

      // リソースの所有者確認（例：ユーザーが自分自身のリソースにアクセスしているか）
      if (resourceId !== userId && req.user.role !== 'admin') {
        return res.status(403).json({
          success: false,
          error: 'このリソースにアクセスする権限がありません'
        });
      }

      next();
    } catch (error) {
      return res.status(500).json({
        success: false,
        error: '認可チェックでエラーが発生しました'
      });
    }
  };
};
```

### レート制限ミドルウェア
```typescript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);

// 基本的なレート制限
const generalLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: 'rl:general:'
  }),
  windowMs: 15 * 60 * 1000, // 15分
  max: 100, // 最大100リクエスト
  message: {
    success: false,
    error: 'リクエストが多すぎます。しばらくしてから再試行してください。'
  },
  standardHeaders: true,
  legacyHeaders: false
});

// API別のレート制限
const createApiLimiter = (windowMs: number, max: number, prefix: string) => {
  return rateLimit({
    store: new RedisStore({
      client: redis,
      prefix: `rl:${prefix}:`
    }),
    windowMs,
    max,
    message: {
      success: false,
      error: `${prefix} APIのリクエスト制限に達しました`
    }
  });
};

// 認証API用（厳しい制限）
const authLimiter = createApiLimiter(15 * 60 * 1000, 5, 'auth');

// データ取得API用（緩い制限）
const dataLimiter = createApiLimiter(15 * 60 * 1000, 1000, 'data');

// ユーザー別カスタムレート制限
const userBasedLimiter = async (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
  const userId = req.user?.id;
  const userRole = req.user?.role;

  if (!userId) {
    return next();
  }

  // ロール別の制限設定
  const limits = {
    admin: { windowMs: 15 * 60 * 1000, max: 1000 },
    premium: { windowMs: 15 * 60 * 1000, max: 500 },
    user: { windowMs: 15 * 60 * 1000, max: 100 }
  };

  const userLimit = limits[userRole as keyof typeof limits] || limits.user;
  
  const key = `rl:user:${userId}`;
  const current = await redis.incr(key);
  
  if (current === 1) {
    await redis.pexpire(key, userLimit.windowMs);
  }
  
  if (current > userLimit.max) {
    return res.status(429).json({
      success: false,
      error: 'ユーザー別リクエスト制限に達しました'
    });
  }
  
  res.set({
    'X-RateLimit-Limit': userLimit.max.toString(),
    'X-RateLimit-Remaining': Math.max(0, userLimit.max - current).toString(),
    'X-RateLimit-Reset': new Date(Date.now() + userLimit.windowMs).toISOString()
  });
  
  next();
};
```

### ログ・監視ミドルウェア
```typescript
import { v4 as uuidv4 } from 'uuid';
import { performance } from 'perf_hooks';

// リクエストトレーシング
const requestTracing = (req: Request, res: Response, next: NextFunction) => {
  req.id = uuidv4();
  req.startTime = performance.now();
  
  // レスポンス完了時の処理
  res.on('finish', () => {
    const endTime = performance.now();
    const duration = endTime - req.startTime;
    
    console.log({
      requestId: req.id,
      method: req.method,
      url: req.url,
      statusCode: res.statusCode,
      duration: `${duration.toFixed(2)}ms`,
      userAgent: req.get('User-Agent'),
      ip: req.ip,
      userId: req.user?.id
    });
  });
  
  next();
};

// アクセスログ
const accessLogger = (req: Request, res: Response, next: NextFunction) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    const logData = {
      timestamp: new Date().toISOString(),
      method: req.method,
      url: req.originalUrl,
      statusCode: res.statusCode,
      responseTime: duration,
      ip: req.ip,
      userAgent: req.get('User-Agent'),
      referer: req.get('Referer'),
      userId: req.user?.id,
      requestId: req.id
    };
    
    // 異常な応答時間の検出
    if (duration > 1000) {
      console.warn('Slow request detected:', logData);
    }
    
    // 分析サービスに送信
    analytics.track('api_request', logData);
  });
  
  next();
};

// ヘルスチェック除外ロガー
const conditionalLogger = (req: Request, res: Response, next: NextFunction) => {
  // ヘルスチェックエンドポイントはログを出力しない
  if (req.path === '/health' || req.path === '/ping') {
    return next();
  }
  
  return accessLogger(req, res, next);
};
```

## GraphQL API パターン

### 基本的なGraphQLスキーマ
```typescript
import { gql } from 'apollo-server-express';
import { GraphQLScalarType } from 'graphql';
import { Kind } from 'graphql/language';

// カスタムスカラー型
const DateTimeType = new GraphQLScalarType({
  name: 'DateTime',
  description: 'ISO 8601形式の日時文字列',
  serialize: (value: Date) => value.toISOString(),
  parseValue: (value: string) => new Date(value),
  parseLiteral: (ast) => {
    if (ast.kind === Kind.STRING) {
      return new Date(ast.value);
    }
    return null;
  }
});

// GraphQLスキーマ定義
const typeDefs = gql`
  scalar DateTime
  
  type User {
    id: ID!
    email: String!
    name: String!
    role: Role!
    profile: Profile
    posts: [Post!]!
    createdAt: DateTime!
    updatedAt: DateTime!
  }
  
  type Profile {
    bio: String
    avatar: String
    phoneNumber: String
  }
  
  type Post {
    id: ID!
    title: String!
    content: String!
    published: Boolean!
    author: User!
    comments: [Comment!]!
    createdAt: DateTime!
    updatedAt: DateTime!
  }
  
  type Comment {
    id: ID!
    content: String!
    author: User!
    post: Post!
    createdAt: DateTime!
  }
  
  enum Role {
    USER
    ADMIN
  }
  
  input CreateUserInput {
    email: String!
    name: String!
    password: String!
    role: Role = USER
    profile: ProfileInput
  }
  
  input ProfileInput {
    bio: String
    avatar: String
    phoneNumber: String
  }
  
  input UpdateUserInput {
    email: String
    name: String
    role: Role
    profile: ProfileInput
  }
  
  input PostFilters {
    published: Boolean
    authorId: ID
    search: String
  }
  
  type Query {
    users(
      first: Int = 10
      after: String
      search: String
      role: Role
    ): UserConnection!
    
    user(id: ID!): User
    
    posts(
      first: Int = 10
      after: String
      filters: PostFilters
    ): PostConnection!
    
    post(id: ID!): Post
  }
  
  type Mutation {
    createUser(input: CreateUserInput!): User!
    updateUser(id: ID!, input: UpdateUserInput!): User!
    deleteUser(id: ID!): Boolean!
    
    createPost(input: CreatePostInput!): Post!
    updatePost(id: ID!, input: UpdatePostInput!): Post!
    deletePost(id: ID!): Boolean!
  }
  
  type Subscription {
    userCreated: User!
    postPublished: Post!
    commentAdded(postId: ID!): Comment!
  }
  
  # ページネーション用の型
  type UserConnection {
    edges: [UserEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }
  
  type UserEdge {
    node: User!
    cursor: String!
  }
  
  type PostConnection {
    edges: [PostEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }
  
  type PostEdge {
    node: Post!
    cursor: String!
  }
  
  type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
  }
`;

// リゾルバー実装
const resolvers = {
  DateTime: DateTimeType,
  
  Query: {
    users: async (parent, args, context) => {
      const { first, after, search, role } = args;
      
      const where: any = {};
      if (search) {
        where.OR = [
          { name: { contains: search, mode: 'insensitive' } },
          { email: { contains: search, mode: 'insensitive' } }
        ];
      }
      if (role) {
        where.role = role;
      }
      
      const cursor = after ? { id: after } : undefined;
      
      const users = await context.prisma.user.findMany({
        where,
        take: first + 1,
        cursor,
        orderBy: { createdAt: 'desc' }
      });
      
      const hasNextPage = users.length > first;
      if (hasNextPage) users.pop();
      
      const edges = users.map(user => ({
        node: user,
        cursor: user.id
      }));
      
      return {
        edges,
        pageInfo: {
          hasNextPage,
          hasPreviousPage: !!after,
          startCursor: edges[0]?.cursor,
          endCursor: edges[edges.length - 1]?.cursor
        },
        totalCount: await context.prisma.user.count({ where })
      };
    },
    
    user: async (parent, { id }, context) => {
      return context.prisma.user.findUnique({
        where: { id }
      });
    }
  },
  
  Mutation: {
    createUser: async (parent, { input }, context) => {
      // バリデーション
      const existingUser = await context.prisma.user.findUnique({
        where: { email: input.email }
      });
      
      if (existingUser) {
        throw new Error('このメールアドレスは既に使用されています');
      }
      
      const hashedPassword = await bcrypt.hash(input.password, 10);
      
      const user = await context.prisma.user.create({
        data: {
          ...input,
          password: hashedPassword
        }
      });
      
      // サブスクリプション通知
      context.pubsub.publish('USER_CREATED', { userCreated: user });
      
      return user;
    }
  },
  
  Subscription: {
    userCreated: {
      subscribe: (parent, args, context) => {
        return context.pubsub.asyncIterator(['USER_CREATED']);
      }
    }
  },
  
  // フィールドリゾルバー
  User: {
    posts: async (parent, args, context) => {
      return context.prisma.post.findMany({
        where: { authorId: parent.id }
      });
    }
  },
  
  Post: {
    author: async (parent, args, context) => {
      return context.prisma.user.findUnique({
        where: { id: parent.authorId }
      });
    },
    
    comments: async (parent, args, context) => {
      return context.prisma.comment.findMany({
        where: { postId: parent.id }
      });
    }
  }
};
```

### GraphQL N+1問題の解決
```typescript
import DataLoader from 'dataloader';

// DataLoaderの実装
const createUserLoader = (prisma: PrismaClient) => {
  return new DataLoader<string, User | null>(async (userIds) => {
    const users = await prisma.user.findMany({
      where: { id: { in: userIds as string[] } }
    });
    
    const userMap = new Map(users.map(user => [user.id, user]));
    return userIds.map(id => userMap.get(id) || null);
  });
};

const createPostsByUserLoader = (prisma: PrismaClient) => {
  return new DataLoader<string, Post[]>(async (userIds) => {
    const posts = await prisma.post.findMany({
      where: { authorId: { in: userIds as string[] } }
    });
    
    const postsByUser = userIds.map(userId => 
      posts.filter(post => post.authorId === userId)
    );
    
    return postsByUser;
  });
};

// コンテキストでDataLoaderを提供
const createContext = ({ req, res }: { req: Request; res: Response }) => {
  return {
    prisma,
    user: req.user,
    loaders: {
      user: createUserLoader(prisma),
      postsByUser: createPostsByUserLoader(prisma)
    }
  };
};

// リゾルバーでDataLoaderを使用
const resolvers = {
  Post: {
    author: async (parent, args, context) => {
      return context.loaders.user.load(parent.authorId);
    }
  },
  
  User: {
    posts: async (parent, args, context) => {
      return context.loaders.postsByUser.load(parent.id);
    }
  }
};
```