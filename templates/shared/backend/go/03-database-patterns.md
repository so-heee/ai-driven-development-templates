# Go データベースパターン

## GORM（ORM）

```go
// モデル定義
type User struct {
    ID        uint      `gorm:"primaryKey"`
    Name      string    `gorm:"size:100;not null"`
    Email     string    `gorm:"uniqueIndex;size:100"`
    CreatedAt time.Time
    UpdatedAt time.Time
    Posts     []Post    `gorm:"foreignKey:UserID"`
}

type Post struct {
    ID        uint   `gorm:"primaryKey"`
    Title     string `gorm:"size:200;not null"`
    Content   string `gorm:"type:text"`
    UserID    uint
    User      User   `gorm:"constraint:OnUpdate:CASCADE,OnDelete:SET NULL;"`
}

// 接続・設定
func initDB() *gorm.DB {
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatal("Failed to connect database")
    }

    // マイグレーション
    db.AutoMigrate(&User{}, &Post{})
    return db
}

// CRUD操作
func createUser(db *gorm.DB, user *User) error {
    return db.Create(user).Error
}

func getUserByID(db *gorm.DB, id uint) (*User, error) {
    var user User
    err := db.Preload("Posts").First(&user, id).Error
    return &user, err
}

func updateUser(db *gorm.DB, id uint, updates map[string]interface{}) error {
    return db.Model(&User{}).Where("id = ?", id).Updates(updates).Error
}
```

## database/sql（標準ライブラリ）

```go
import (
    "database/sql"
    _ "github.com/lib/pq"
)

type UserRepository struct {
    db *sql.DB
}

func (r *UserRepository) Create(ctx context.Context, user *User) error {
    query := `INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id`
    err := r.db.QueryRowContext(ctx, query, user.Name, user.Email).Scan(&user.ID)
    return err
}

func (r *UserRepository) GetByID(ctx context.Context, id int) (*User, error) {
    query := `SELECT id, name, email FROM users WHERE id = $1`
    row := r.db.QueryRowContext(ctx, query, id)

    var user User
    err := row.Scan(&user.ID, &user.Name, &user.Email)
    if err == sql.ErrNoRows {
        return nil, nil
    }
    return &user, err
}
```

## トランザクション

```go
func transferFunds(db *gorm.DB, fromID, toID uint, amount float64) error {
    return db.Transaction(func(tx *gorm.DB) error {
        // 残高確認
        var fromAccount Account
        if err := tx.First(&fromAccount, fromID).Error; err != nil {
            return err
        }

        if fromAccount.Balance < amount {
            return errors.New("insufficient funds")
        }

        // 残高更新
        if err := tx.Model(&fromAccount).Update("balance", fromAccount.Balance-amount).Error; err != nil {
            return err
        }

        var toAccount Account
        if err := tx.First(&toAccount, toID).Error; err != nil {
            return err
        }

        return tx.Model(&toAccount).Update("balance", toAccount.Balance+amount).Error
    })
}
```

## MongoDB

```go
import (
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

type UserRepo struct {
    collection *mongo.Collection
}

func (r *UserRepo) Create(ctx context.Context, user *User) error {
    result, err := r.collection.InsertOne(ctx, user)
    if err != nil {
        return err
    }
    user.ID = result.InsertedID.(primitive.ObjectID)
    return nil
}

func (r *UserRepo) FindByEmail(ctx context.Context, email string) (*User, error) {
    var user User
    filter := bson.D{{"email", email}}
    err := r.collection.FindOne(ctx, filter).Decode(&user)
    return &user, err
}
```
