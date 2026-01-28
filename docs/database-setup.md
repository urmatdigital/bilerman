# Database Setup: BILERMAN.KG

## Подключение к Neon PostgreSQL

### Connection String
```
postgresql://neondb_owner:npg_RJqe61IFpEyO@ep-young-wildflower-agt7kso0-pooler.c-2.eu-central-1.aws.neon.tech/neondb?sslmode=require&channel_binding=require
```

### Environment Variables

**Backend `.env`**:
```env
DATABASE_URL=postgresql://neondb_owner:npg_RJqe61IFpEyO@ep-young-wildflower-agt7kso0-pooler.c-2.eu-central-1.aws.neon.tech/neondb?sslmode=require&channel_binding=require

# Для SQLAlchemy (async)
DATABASE_URL_ASYNC=postgresql+asyncpg://neondb_owner:npg_RJqe61IFpEyO@ep-young-wildflower-agt7kso0-pooler.c-2.eu-central-1.aws.neon.tech/neondb?ssl=require

# JWT
JWT_SECRET_KEY=your-super-secret-key-change-in-production
JWT_ALGORITHM=HS256
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=15
JWT_REFRESH_TOKEN_EXPIRE_DAYS=30

# Mapbox
MAPBOX_ACCESS_TOKEN=pk.your-mapbox-token

# App
APP_NAME=BILERMAN.KG
APP_ENV=development
```

---

## Начальная миграция

### 001_initial_schema.sql

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- =====================
-- USERS
-- =====================
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    phone VARCHAR(20),
    role VARCHAR(20) DEFAULT 'client' CHECK (role IN ('client', 'partner', 'staff')),
    avatar_url TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    is_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);

-- =====================
-- PARTNER PROFILES
-- =====================
CREATE TABLE partner_profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE UNIQUE,

    -- Тип партнёра
    business_type VARCHAR(20) CHECK (business_type IN ('individual', 'ip', 'company')),

    -- Данные компании/ИП
    company_name VARCHAR(255),
    inn VARCHAR(20),
    legal_address TEXT,
    contact_person VARCHAR(255),

    -- Банковские реквизиты
    bank_name VARCHAR(255),
    bank_bik VARCHAR(20),
    bank_account VARCHAR(30),

    -- Альтернативные способы выплат
    mbank_phone VARCHAR(20),
    elsom_wallet VARCHAR(50),

    -- Документы (JSON с URL файлов)
    documents JSONB DEFAULT '{}',

    -- Статус верификации
    is_verified BOOLEAN DEFAULT FALSE,
    verified_at TIMESTAMP WITH TIME ZONE,
    verification_notes TEXT,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_partner_profiles_user ON partner_profiles(user_id);

-- =====================
-- PROPERTIES
-- =====================
CREATE TABLE properties (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID REFERENCES users(id) ON DELETE CASCADE,

    -- Основная информация
    title VARCHAR(255) NOT NULL,
    description TEXT,
    description_ky TEXT, -- Кыргызский

    -- Тип и локация
    property_type VARCHAR(50) CHECK (property_type IN ('apartment', 'house', 'cottage', 'aframe', 'room', 'hostel')),
    address TEXT,
    city VARCHAR(100),
    region VARCHAR(100),
    latitude DECIMAL(10, 8),
    longitude DECIMAL(11, 8),

    -- Характеристики
    max_guests INT DEFAULT 1,
    bedrooms INT DEFAULT 1,
    beds INT DEFAULT 1,
    bathrooms INT DEFAULT 1,
    area_sqm INT, -- Площадь в кв.м

    -- Удобства (массив строк)
    amenities JSONB DEFAULT '[]',

    -- Фотографии (массив URL)
    photos JSONB DEFAULT '[]',

    -- Цены
    price_per_night DECIMAL(10, 2) NOT NULL,
    weekend_price DECIMAL(10, 2),
    weekly_discount INT, -- Процент скидки
    min_nights INT DEFAULT 1,
    max_nights INT DEFAULT 365,

    -- Политики
    cancellation_policy VARCHAR(20) DEFAULT 'moderate' CHECK (cancellation_policy IN ('flexible', 'moderate', 'strict')),
    check_in_time TIME DEFAULT '14:00',
    check_out_time TIME DEFAULT '12:00',
    house_rules TEXT,

    -- Статус и модерация
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'active', 'rejected', 'blocked', 'draft')),
    rejection_reason TEXT,

    -- Рейтинг
    rating DECIMAL(2, 1) DEFAULT 0,
    reviews_count INT DEFAULT 0,

    -- Метаданные
    is_featured BOOLEAN DEFAULT FALSE,
    views_count INT DEFAULT 0,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_properties_owner ON properties(owner_id);
CREATE INDEX idx_properties_city ON properties(city);
CREATE INDEX idx_properties_type ON properties(property_type);
CREATE INDEX idx_properties_status ON properties(status);
CREATE INDEX idx_properties_price ON properties(price_per_night);
CREATE INDEX idx_properties_location ON properties USING GIST (
    ST_SetSRID(ST_MakePoint(longitude, latitude), 4326)
);

-- =====================
-- BOOKINGS
-- =====================
CREATE TABLE bookings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Связи
    property_id UUID REFERENCES properties(id) ON DELETE SET NULL,
    client_id UUID REFERENCES users(id) ON DELETE SET NULL,

    -- Даты
    check_in DATE NOT NULL,
    check_out DATE NOT NULL,
    nights INT GENERATED ALWAYS AS (check_out - check_in) STORED,

    -- Гости
    guests INT DEFAULT 1,

    -- Финансы
    base_price DECIMAL(10, 2) NOT NULL,
    service_fee DECIMAL(10, 2) NOT NULL,
    total_price DECIMAL(10, 2) NOT NULL,

    -- Статус
    status VARCHAR(20) DEFAULT 'pending_payment' CHECK (status IN (
        'pending_payment', 'confirmed', 'checked_in', 'checked_out',
        'completed', 'cancelled', 'dispute'
    )),

    -- Договор
    contract_url TEXT,

    -- Дополнительно
    guest_notes TEXT,
    host_notes TEXT,
    cancelled_by VARCHAR(20),
    cancelled_at TIMESTAMP WITH TIME ZONE,
    cancellation_reason TEXT,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_bookings_property ON bookings(property_id);
CREATE INDEX idx_bookings_client ON bookings(client_id);
CREATE INDEX idx_bookings_dates ON bookings(check_in, check_out);
CREATE INDEX idx_bookings_status ON bookings(status);

-- =====================
-- PAYMENTS
-- =====================
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id UUID REFERENCES bookings(id) ON DELETE SET NULL,

    -- Суммы
    amount DECIMAL(10, 2) NOT NULL,
    commission DECIMAL(10, 2) NOT NULL, -- 10%
    partner_amount DECIMAL(10, 2) NOT NULL, -- 90%

    -- Метод оплаты
    method VARCHAR(20) CHECK (method IN ('mbank', 'odengi', 'elsom', 'card', 'cash')),

    -- Статус
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN (
        'pending', 'processing', 'escrow', 'released', 'refunded', 'failed'
    )),

    -- Временные метки
    escrow_at TIMESTAMP WITH TIME ZONE,
    released_at TIMESTAMP WITH TIME ZONE,
    refunded_at TIMESTAMP WITH TIME ZONE,

    -- Внешние ID
    external_id VARCHAR(100),
    external_data JSONB,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_payments_booking ON payments(booking_id);
CREATE INDEX idx_payments_status ON payments(status);

-- =====================
-- PAYOUTS
-- =====================
CREATE TABLE payouts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_id UUID REFERENCES users(id) ON DELETE SET NULL,

    amount DECIMAL(10, 2) NOT NULL,
    method VARCHAR(20),
    account_details JSONB,

    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN (
        'pending', 'processing', 'completed', 'failed'
    )),

    processed_at TIMESTAMP WITH TIME ZONE,
    external_id VARCHAR(100),
    notes TEXT,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_payouts_partner ON payouts(partner_id);
CREATE INDEX idx_payouts_status ON payouts(status);

-- =====================
-- REVIEWS
-- =====================
CREATE TABLE reviews (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    booking_id UUID REFERENCES bookings(id) ON DELETE SET NULL UNIQUE,
    property_id UUID REFERENCES properties(id) ON DELETE CASCADE,
    client_id UUID REFERENCES users(id) ON DELETE SET NULL,

    -- Рейтинги
    rating INT CHECK (rating >= 1 AND rating <= 5),
    cleanliness INT CHECK (cleanliness >= 1 AND cleanliness <= 5),
    accuracy INT CHECK (accuracy >= 1 AND accuracy <= 5),
    location INT CHECK (location >= 1 AND location <= 5),
    communication INT CHECK (communication >= 1 AND communication <= 5),

    -- Текст
    comment TEXT,

    -- Ответ хозяина
    host_response TEXT,
    host_response_at TIMESTAMP WITH TIME ZONE,

    -- Модерация
    is_approved BOOLEAN DEFAULT TRUE,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_reviews_property ON reviews(property_id);
CREATE INDEX idx_reviews_client ON reviews(client_id);

-- =====================
-- CONTRACTS
-- =====================
CREATE TABLE contracts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id UUID REFERENCES bookings(id) ON DELETE SET NULL UNIQUE,

    -- Снимки данных на момент создания
    partner_data JSONB NOT NULL,
    client_data JSONB NOT NULL,
    booking_data JSONB NOT NULL,
    property_data JSONB NOT NULL,

    -- PDF
    pdf_url TEXT,

    -- Подписи
    partner_signed_at TIMESTAMP WITH TIME ZONE,
    client_signed_at TIMESTAMP WITH TIME ZONE,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_contracts_booking ON contracts(booking_id);

-- =====================
-- PROPERTY AVAILABILITY
-- =====================
CREATE TABLE property_availability (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID REFERENCES properties(id) ON DELETE CASCADE,

    date DATE NOT NULL,
    status VARCHAR(20) DEFAULT 'available' CHECK (status IN ('available', 'booked', 'blocked')),
    booking_id UUID REFERENCES bookings(id) ON DELETE SET NULL,
    custom_price DECIMAL(10, 2),

    UNIQUE(property_id, date)
);

CREATE INDEX idx_availability_property_date ON property_availability(property_id, date);

-- =====================
-- FAVORITES
-- =====================
CREATE TABLE favorites (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    property_id UUID REFERENCES properties(id) ON DELETE CASCADE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    UNIQUE(user_id, property_id)
);

CREATE INDEX idx_favorites_user ON favorites(user_id);

-- =====================
-- REFRESH TOKENS
-- =====================
CREATE TABLE refresh_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) NOT NULL,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    revoked_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_refresh_tokens_user ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_hash ON refresh_tokens(token_hash);

-- =====================
-- FUNCTIONS
-- =====================

-- Обновление updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Триггеры
CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER update_properties_updated_at
    BEFORE UPDATE ON properties
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER update_bookings_updated_at
    BEFORE UPDATE ON bookings
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER update_partner_profiles_updated_at
    BEFORE UPDATE ON partner_profiles
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- Обновление рейтинга при добавлении отзыва
CREATE OR REPLACE FUNCTION update_property_rating()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE properties
    SET
        rating = (
            SELECT ROUND(AVG(rating)::numeric, 1)
            FROM reviews
            WHERE property_id = NEW.property_id AND is_approved = TRUE
        ),
        reviews_count = (
            SELECT COUNT(*)
            FROM reviews
            WHERE property_id = NEW.property_id AND is_approved = TRUE
        )
    WHERE id = NEW.property_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_rating_on_review
    AFTER INSERT OR UPDATE ON reviews
    FOR EACH ROW EXECUTE FUNCTION update_property_rating();

-- =====================
-- SEED DATA
-- =====================

-- Тестовый staff пользователь
INSERT INTO users (email, password_hash, first_name, last_name, role, is_verified)
VALUES (
    'admin@bilerman.kg',
    '$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/X4.G1vO3FYHvGpTHy', -- password: admin123
    'Admin',
    'Bilerman',
    'staff',
    TRUE
);

-- Тестовые города
-- (Будут использоваться для фильтров)
```

---

## Применение миграции

```bash
# Подключение к Neon
psql "postgresql://neondb_owner:npg_RJqe61IFpEyO@ep-young-wildflower-agt7kso0-pooler.c-2.eu-central-1.aws.neon.tech/neondb?sslmode=require"

# Применение миграции
\i 001_initial_schema.sql
```

Или через Python:

```python
import asyncpg
import asyncio

async def run_migration():
    conn = await asyncpg.connect(
        "postgresql://neondb_owner:npg_RJqe61IFpEyO@ep-young-wildflower-agt7kso0-pooler.c-2.eu-central-1.aws.neon.tech/neondb?ssl=require"
    )

    with open('001_initial_schema.sql', 'r') as f:
        sql = f.read()

    await conn.execute(sql)
    await conn.close()
    print("Migration completed!")

asyncio.run(run_migration())
```

---

## Проверка подключения

```python
# test_connection.py
import asyncio
import asyncpg

async def test():
    conn = await asyncpg.connect(
        "postgresql://neondb_owner:npg_RJqe61IFpEyO@ep-young-wildflower-agt7kso0-pooler.c-2.eu-central-1.aws.neon.tech/neondb?ssl=require"
    )
    result = await conn.fetchval("SELECT version()")
    print(f"Connected to: {result}")
    await conn.close()

asyncio.run(test())
```

---

**Подготовлено**: 2026-01-28
