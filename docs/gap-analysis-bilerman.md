# Gap Analysis: KONOK → BILERMAN.KG

**Версия**: 1.0
**Дата**: 2026-01-28
**Цель**: Детальный анализ изменений в коде для трансформации

---

## 1. Обзор изменений

| Категория | Файлов затронуто | Сложность | Приоритет |
|-----------|------------------|-----------|-----------|
| Брендинг | ~20 | Низкая | P0 |
| Аутентификация (OTP) | ~15 | Средняя | P0 |
| Платежи (Эскроу) | ~25 | Высокая | P0 |
| База данных | ~10 | Средняя | P0 |
| Бизнес-логика | ~30 | Средняя | P1 |
| Удаление лишнего | ~40 | Низкая | P2 |
| UI/UX адаптация | ~50 | Средняя | P1 |

---

## 2. Брендинг

### 2.1 Изменения в конфигурации

**Файл**: `frontend/src/config/index.ts`
```typescript
// БЫЛО (konok)
export const APP_NAME = 'KonOk';
export const APP_URL = 'https://konok.kg';

// СТАЛО (bilerman)
export const APP_NAME = 'BILERMAN.KG';
export const APP_URL = 'https://bilerman.kg';
export const APP_TAGLINE = 'Твои деньги защищены до заселения';
```

**Файл**: `frontend/index.html`
```html
<!-- Изменить title, meta tags, favicon -->
<title>BILERMAN.KG — Безопасное бронирование жилья</title>
```

### 2.2 Цветовая схема

**Файл**: `frontend/tailwind.config.js`
```javascript
// БЫЛО
colors: {
  primary: { /* текущие цвета */ }
}

// СТАЛО (из BOOGUARD брендбука)
colors: {
  primary: {
    50: '#fff7ed',
    100: '#ffedd5',
    500: '#f97316', // Оранжевый (акцент)
    600: '#ea580c',
    700: '#c2410c',
  },
  secondary: {
    500: '#3b82f6', // Синий
    600: '#2563eb',
    700: '#1d4ed8',
  },
  navy: {
    800: '#1e3a5f', // Тёмно-синий (фон)
    900: '#172554',
  }
}
```

### 2.3 Логотип и assets

**Действия:**
- [ ] Создать новый логотип BILERMAN.KG
- [ ] Обновить favicon (`frontend/public/favicon.ico`)
- [ ] Обновить PWA иконки (`frontend/public/icons/`)
- [ ] Обновить Open Graph изображения

---

## 3. Аутентификация (OTP по SMS)

### 3.1 Backend изменения

**Новый файл**: `backend/sms/provider.py`
```python
# Интеграция с SMS провайдером (Nikita.kg)
class SMSProvider:
    async def send_otp(self, phone: str, code: str) -> bool:
        # Реализация отправки SMS
        pass
```

**Изменить**: `backend/auth/router.py`
```python
# БЫЛО: Email + password
@router.post("/register")
async def register(user: UserRegister):
    # email, password logic

# СТАЛО: Phone + OTP
@router.post("/request-otp")
async def request_otp(phone: str):
    code = generate_otp()  # 6 цифр
    await redis.setex(f"otp:{phone}", 300, code)  # 5 минут
    await sms_provider.send_otp(phone, code)
    return {"message": "OTP sent"}

@router.post("/verify-otp")
async def verify_otp(phone: str, code: str):
    stored_code = await redis.get(f"otp:{phone}")
    if stored_code == code:
        user = await get_or_create_user(phone)
        tokens = create_tokens(user)
        return tokens
    raise HTTPException(401, "Invalid OTP")
```

**Изменить**: `backend/models/user.py`
```python
# БЫЛО
class User(Base):
    email = Column(String, unique=True, index=True)
    hashed_password = Column(String)

# СТАЛО
class User(Base):
    phone = Column(String(20), unique=True, index=True)  # +996XXXXXXXXX
    email = Column(String, nullable=True)  # Опционально
    # Убрать hashed_password для OTP-only auth
```

### 3.2 Frontend изменения

**Изменить**: `frontend/src/pages/public/Login.tsx`
```tsx
// БЫЛО: Email + Password форма
// СТАЛО: Phone + OTP форма

const [step, setStep] = useState<'phone' | 'otp'>('phone');
const [phone, setPhone] = useState('');

// Step 1: Ввод телефона
// Step 2: Ввод OTP кода
```

**Изменить**: `frontend/src/services/auth.service.ts`
```typescript
// БЫЛО
login(email: string, password: string)
register(email: string, password: string, name: string)

// СТАЛО
requestOTP(phone: string): Promise<void>
verifyOTP(phone: string, code: string): Promise<AuthResponse>
```

**Изменить**: `frontend/src/store/auth.store.ts`
```typescript
// Обновить типы и методы под OTP
interface AuthState {
  user: User | null;
  isAuthenticated: boolean;
  requestOTP: (phone: string) => Promise<void>;
  verifyOTP: (phone: string, code: string) => Promise<void>;
}
```

### 3.3 База данных

**Миграция**: `database/migrations/001_auth_otp.sql`
```sql
-- Изменить таблицу users
ALTER TABLE users
  ADD COLUMN phone VARCHAR(20) UNIQUE,
  ALTER COLUMN email DROP NOT NULL,
  DROP COLUMN hashed_password;

-- Добавить индекс
CREATE INDEX idx_users_phone ON users(phone);

-- Таблица для OTP (или использовать Redis)
CREATE TABLE otp_codes (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  phone VARCHAR(20) NOT NULL,
  code VARCHAR(6) NOT NULL,
  expires_at TIMESTAMP NOT NULL,
  used BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 4. Платежи и Эскроу

### 4.1 Новые модели данных

**Новый файл**: `backend/models/payment.py`
```python
class PaymentMethod(str, Enum):
    MBANK = "mbank"
    ODENGI = "odengi"
    ELSOM = "elsom"
    CARD = "card"

class PaymentStatus(str, Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    ESCROW = "escrow"  # Деньги на эскроу
    RELEASED = "released"  # Выплачено партнёру
    REFUNDED = "refunded"
    FAILED = "failed"

class Payment(Base):
    __tablename__ = "payments"

    id = Column(UUID, primary_key=True)
    booking_id = Column(UUID, ForeignKey("bookings.id"))
    amount = Column(Numeric(10, 2))  # Полная сумма
    commission = Column(Numeric(10, 2))  # 10%
    partner_amount = Column(Numeric(10, 2))  # 90%
    method = Column(Enum(PaymentMethod))
    status = Column(Enum(PaymentStatus), default=PaymentStatus.PENDING)
    escrow_at = Column(DateTime)  # Когда поступило на эскроу
    released_at = Column(DateTime)  # Когда выплачено партнёру
    external_id = Column(String)  # ID транзакции в платёжной системе
```

**Новый файл**: `backend/models/payout.py`
```python
class Payout(Base):
    __tablename__ = "payouts"

    id = Column(UUID, primary_key=True)
    partner_id = Column(UUID, ForeignKey("users.id"))
    payment_id = Column(UUID, ForeignKey("payments.id"))
    amount = Column(Numeric(10, 2))
    method = Column(String)  # mbank, elsom, bank_transfer
    account_details = Column(JSON)  # Реквизиты
    status = Column(String)  # pending, processing, completed, failed
    processed_at = Column(DateTime)
```

### 4.2 Интеграции платёжных систем

**Новая директория**: `backend/payments/`
```
backend/payments/
├── __init__.py
├── base.py          # Абстрактный класс PaymentProvider
├── mbank.py         # MBank интеграция
├── odengi.py        # O!Dengi интеграция
├── elsom.py         # Элсом интеграция
├── escrow.py        # Логика эскроу
└── router.py        # API endpoints
```

**Файл**: `backend/payments/mbank.py`
```python
class MBankProvider(PaymentProvider):
    """
    Интеграция с MBank API
    - Генерация QR-кода для оплаты
    - Webhook для подтверждения платежа
    - Выплаты на счёт MBank
    """

    async def create_payment(self, amount: Decimal, order_id: str) -> PaymentRequest:
        # Создать запрос на оплату, получить QR
        pass

    async def verify_payment(self, external_id: str) -> bool:
        # Проверить статус платежа
        pass

    async def payout(self, amount: Decimal, account: str) -> PayoutResult:
        # Выплата на счёт партнёра
        pass
```

### 4.3 Эскроу логика

**Файл**: `backend/payments/escrow.py`
```python
class EscrowService:
    """
    Управление эскроу-счётом
    """

    async def hold_payment(self, payment_id: UUID) -> None:
        """Зафиксировать платёж на эскроу после успешной оплаты"""
        payment = await self.get_payment(payment_id)
        payment.status = PaymentStatus.ESCROW
        payment.escrow_at = datetime.utcnow()
        await self.save(payment)

    async def release_to_partner(self, booking_id: UUID) -> None:
        """Выплатить партнёру после check-in"""
        booking = await self.get_booking(booking_id)
        payment = booking.payment

        # Создать выплату
        payout = Payout(
            partner_id=booking.property.owner_id,
            payment_id=payment.id,
            amount=payment.partner_amount,  # 90%
            method=partner.payout_method,
            account_details=partner.payout_details,
        )

        # Обработать выплату
        await self.process_payout(payout)

        payment.status = PaymentStatus.RELEASED
        payment.released_at = datetime.utcnow()

    async def refund_to_client(self, payment_id: UUID, amount: Decimal) -> None:
        """Возврат клиенту (полный или частичный)"""
        pass
```

### 4.4 Frontend: страница оплаты

**Новый файл**: `frontend/src/pages/guest/Checkout.tsx`
```tsx
// Страница оформления и оплаты бронирования
// 1. Сводка бронирования
// 2. Выбор способа оплаты
// 3. QR-код или редирект на платёжную систему
// 4. Ожидание подтверждения
// 5. Успех / Ошибка
```

### 4.5 База данных

**Миграция**: `database/migrations/002_payments.sql`
```sql
-- Таблица платежей
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    booking_id UUID REFERENCES bookings(id),
    amount DECIMAL(10,2) NOT NULL,
    commission DECIMAL(10,2) NOT NULL,
    partner_amount DECIMAL(10,2) NOT NULL,
    method VARCHAR(20) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    escrow_at TIMESTAMP,
    released_at TIMESTAMP,
    external_id VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Таблица выплат партнёрам
CREATE TABLE payouts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    partner_id UUID REFERENCES users(id),
    payment_id UUID REFERENCES payments(id),
    amount DECIMAL(10,2) NOT NULL,
    method VARCHAR(20) NOT NULL,
    account_details JSONB,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    processed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Компенсационный фонд
CREATE TABLE compensation_fund (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    balance DECIMAL(12,2) NOT NULL DEFAULT 0,
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Транзакции фонда
CREATE TABLE fund_transactions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    type VARCHAR(20) NOT NULL, -- 'contribution', 'compensation'
    amount DECIMAL(10,2) NOT NULL,
    booking_id UUID REFERENCES bookings(id),
    description TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 5. Бизнес-логика: Отмены и неустойки

### 5.1 Модель политики отмены

**Изменить**: `backend/models/property.py`
```python
class CancellationPolicy(str, Enum):
    FLEXIBLE = "flexible"
    MODERATE = "moderate"
    STRICT = "strict"

class Property(Base):
    # ... существующие поля ...
    cancellation_policy = Column(Enum(CancellationPolicy), default=CancellationPolicy.MODERATE)
```

### 5.2 Сервис отмены

**Новый файл**: `backend/bookings/cancellation.py`
```python
class CancellationService:
    POLICIES = {
        CancellationPolicy.FLEXIBLE: [
            (3, 100),   # > 3 дней = 100% возврат
            (1, 50),    # 1-3 дня = 50% возврат
            (0, 0),     # < 1 дня = 0% возврат
        ],
        CancellationPolicy.MODERATE: [
            (14, 100),
            (7, 50),
            (0, 0),
        ],
        CancellationPolicy.STRICT: [
            (30, 100),
            (14, 50),
            (0, 0),
        ],
    }

    PARTNER_PENALTIES = [
        (14, 0),    # > 14 дней = без штрафа
        (7, 10),    # 7-14 дней = 10% штраф
        (3, 30),    # 3-7 дней = 30% штраф
        (1, 50),    # 1-3 дня = 50% штраф
        (0, 100),   # день заезда = 100% штраф
    ]

    async def cancel_by_client(self, booking_id: UUID) -> CancellationResult:
        """Отмена клиентом"""
        booking = await self.get_booking(booking_id)
        days_before = (booking.check_in - date.today()).days

        policy = self.POLICIES[booking.property.cancellation_policy]
        refund_percent = self._get_refund_percent(days_before, policy)

        refund_amount = booking.total_price * refund_percent / 100
        await self.escrow.refund_to_client(booking.payment_id, refund_amount)

        booking.status = BookingStatus.CANCELLED
        booking.cancelled_by = "client"
        booking.refund_amount = refund_amount

        return CancellationResult(refund_amount=refund_amount)

    async def cancel_by_partner(self, booking_id: UUID) -> CancellationResult:
        """Отмена партнёром (со штрафом)"""
        booking = await self.get_booking(booking_id)
        days_before = (booking.check_in - date.today()).days

        # Клиенту всегда 100% возврат
        await self.escrow.refund_to_client(booking.payment_id, booking.total_price)

        # Штраф партнёру
        penalty_percent = self._get_penalty_percent(days_before)
        penalty_amount = booking.total_price * penalty_percent / 100

        if penalty_amount > 0:
            await self.apply_partner_penalty(booking.property.owner_id, penalty_amount)

        booking.status = BookingStatus.CANCELLED
        booking.cancelled_by = "partner"
        booking.penalty_amount = penalty_amount

        return CancellationResult(penalty_amount=penalty_amount)
```

---

## 6. Удаление лишнего функционала

### 6.1 Удалить интеграции (для MVP)

**Удалить директории:**
```
backend/integration_service/integrations/booking_com/
backend/integration_service/integrations/airbnb/
backend/integration_service/integrations/local_kg/
```

**Удалить из фронтенда:**
- Страницы интеграций
- Компоненты синхронизации
- Типы и сервисы для внешних платформ

### 6.2 Упростить роли

**Было:**
- guest, owner, cleaner, admin

**Стало (MVP):**
- client (клиент)
- partner (арендодатель)
- admin

**Удалить:**
- Всю логику клинеров (`/pages/cleaner/`, `/models/cleaner.py`)
- Задачи на уборку
- Связанные таблицы в БД

### 6.3 Удалить микросервисы

**Оставить monolith для MVP:**
```
# Удалить или отключить:
backend/communication_service/  # Заменить на простые SMS
backend/pricing_analytics_service/  # Не нужно для MVP
backend/copilot/  # AI-чаты не приоритет
```

---

## 7. Изменения в UI/UX

### 7.1 Landing Page

**Файл**: `frontend/src/pages/public/Landing.tsx`

Адаптировать под концепцию BOOGUARD:
- Hero секция: "Бронируй безопасно" + УТП эскроу
- Как это работает (4 шага)
- Для клиентов / Для партнёров
- Отзывы
- CTA: "Найти жильё" / "Стать партнёром"

### 7.2 Страница партнёра

**Файл**: `frontend/src/pages/public/Partners.tsx`

Контент из BOOGUARD_КП_Партнёры.pdf:
- Проблемы (500+ звонков, no-show)
- Решение (бесплатная CRM)
- Как это работает
- Условия (10% комиссия)
- Защита от отмен
- CTA: "Подключиться"

### 7.3 Dashboard партнёра

**Файл**: `frontend/src/pages/owner/OwnerDashboard.tsx`

Добавить:
- Баланс (доступно к выводу)
- Ожидает выплаты (после check-in)
- Кнопка "Вывести средства"
- История выплат

### 7.4 Страница объекта

**Файл**: `frontend/src/pages/public/PropertyDetail.tsx`

Добавить:
- Значок "Защита BILERMAN" (эскроу)
- Политика отмены (видимая)
- Кнопка "Забронировать" → Checkout

---

## 8. Переименования и рефакторинг

### 8.1 Терминология

| Было (konok) | Стало (bilerman) |
|--------------|----------------|
| Guest | Client (Клиент) |
| Owner | Partner (Партнёр) |
| Property | Object (Объект) или Property |
| Booking | Booking (Бронирование) |
| Unit | — (убрать, один объект = один листинг) |

### 8.2 Файлы для переименования

```
frontend/src/pages/guest/ → frontend/src/pages/client/
frontend/src/pages/owner/ → frontend/src/pages/partner/
backend/models/user.py: role='guest' → role='client'
backend/models/user.py: role='owner' → role='partner'
```

---

## 9. Тестирование

### 9.1 Критические сценарии для тестирования

1. **Регистрация по SMS**
   - Отправка OTP
   - Верификация кода
   - Повторная отправка
   - Истечение срока

2. **Бронирование с оплатой**
   - Создание бронирования
   - Оплата через MBank
   - Webhook подтверждения
   - Статус эскроу

3. **Check-in и выплата**
   - Подтверждение заселения
   - Расчёт комиссии
   - Выплата партнёру
   - Обновление баланса

4. **Отмена клиентом**
   - Расчёт возврата по политике
   - Возврат средств
   - Уведомления

5. **Отмена партнёром**
   - 100% возврат клиенту
   - Расчёт штрафа
   - Применение неустойки

---

## 10. План миграции данных

### 10.1 Существующие данные konok

Если есть production данные:
1. Бэкап текущей БД
2. Миграция схемы
3. Маппинг ролей (guest→client, owner→partner)
4. Добавление обязательных полей (phone)
5. Очистка устаревших данных

### 10.2 Новая БД (если с нуля)

Применить все миграции:
```bash
psql -d bilerman -f database/migrations/001_auth_otp.sql
psql -d bilerman -f database/migrations/002_payments.sql
psql -d bilerman -f database/migrations/003_cancellation.sql
```

---

## 11. Чеклист готовности к MVP

### Backend:
- [ ] SMS провайдер подключён
- [ ] OTP аутентификация работает
- [ ] MBank интеграция работает
- [ ] Эскроу логика реализована
- [ ] Отмены с расчётом возврата
- [ ] Выплаты партнёрам
- [ ] API задокументировано

### Frontend:
- [ ] Брендинг обновлён
- [ ] Регистрация/вход по телефону
- [ ] Checkout с оплатой
- [ ] Dashboard партнёра с балансом
- [ ] Политики отмены отображаются
- [ ] PWA работает

### Инфраструктура:
- [ ] Домен bilerman.kg настроен
- [ ] SSL сертификат
- [ ] Production окружение
- [ ] Мониторинг (Sentry)
- [ ] Бэкапы БД

### Юридическое:
- [ ] ОсОО зарегистрировано
- [ ] Договор оферты
- [ ] Политика конфиденциальности
- [ ] Договор с платёжными системами

---

**Документ подготовлен**: 2026-01-28
**Следующий шаг**: Epics & Stories для разработки
