# TypeScript 認証実装

## JWT認証

```typescript
import jwt from 'jsonwebtoken'
import bcrypt from 'bcryptjs'

interface JwtPayload {
  userId: number
  email: string
  role: string
}

class AuthService {
  private readonly JWT_SECRET = process.env.JWT_SECRET!

  generateToken(payload: JwtPayload): string {
    return jwt.sign(payload, this.JWT_SECRET, {
      expiresIn: '24h',
      issuer: 'myapp',
    })
  }

  verifyToken(token: string): JwtPayload {
    return jwt.verify(token, this.JWT_SECRET) as JwtPayload
  }

  async hashPassword(password: string): Promise<string> {
    return bcrypt.hash(password, 12)
  }

  async comparePassword(password: string, hash: string): Promise<boolean> {
    return bcrypt.compare(password, hash)
  }
}

// Express middleware
export const authMiddleware = (req: Request, res: Response, next: NextFunction) => {
  const token = req.headers.authorization?.split(' ')[1]

  if (!token) {
    return res.status(401).json({ error: 'No token provided' })
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as JwtPayload
    req.user = decoded
    next()
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' })
  }
}
```

## Passport.js

```typescript
import passport from 'passport'
import { Strategy as LocalStrategy } from 'passport-local'
import { Strategy as JwtStrategy, ExtractJwt } from 'passport-jwt'

// Local Strategy
passport.use(
  new LocalStrategy({ usernameField: 'email' }, async (email, password, done) => {
    try {
      const user = await userService.findByEmail(email)
      if (!user || !(await authService.comparePassword(password, user.password))) {
        return done(null, false)
      }
      return done(null, user)
    } catch (error) {
      return done(error)
    }
  })
)

// JWT Strategy
passport.use(
  new JwtStrategy(
    {
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: process.env.JWT_SECRET,
    },
    async (payload, done) => {
      try {
        const user = await userService.findById(payload.userId)
        return done(null, user)
      } catch (error) {
        return done(error)
      }
    }
  )
)
```
