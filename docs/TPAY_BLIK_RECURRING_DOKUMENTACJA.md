# Dokumentacja Techniczna Integracji TPay BLIK Recurring

## Informacje o Sprzedawcy

| Dane | Wartość |
|------|---------|
| **Nazwa firmy** | FENIX HORSE SP. Z O.O. |
| **NIP** | 7773026028 |
| **Adres** | ul. Klonowa 11, 62-095 Murowana Goślina |
| **Strona WWW** | https://gonative.pl |
| **E-mail kontaktowy** | kontakt@gonative.pl |

---

## 1. Opis Integracji

### 1.1 Model Płatności
- **Typ:** BLIK Recurring (Model O)
- **groupId:** 150
- **Cel:** Automatyczne pobieranie płatności cyklicznych za subskrypcje karmy dla psów

### 1.2 Przypadek Użycia
Sklep internetowy GO NATIVE oferuje subskrypcję premium karmy dla psów z automatycznym odnawianiem. Klient wybiera produkty, częstotliwość dostawy (1-16 tygodni) i autoryzuje płatność cykliczną BLIK. System automatycznie pobiera płatności przed każdą dostawą.

---

## 2. Przepływ Płatności

### 2.1 Pierwsza Płatność (Rejestracja Aliasu)

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Klient        │     │   Sklep         │     │   TPay API      │
│   (Frontend)    │     │   (Backend)     │     │                 │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         │ 1. Wybór subskrypcji  │                       │
         │ i płatności TPay      │                       │
         │──────────────────────>│                       │
         │                       │                       │
         │                       │ 2. POST /transactions │
         │                       │ (recursive: true)     │
         │                       │──────────────────────>│
         │                       │                       │
         │                       │ 3. transactionPaymentUrl
         │                       │<──────────────────────│
         │                       │                       │
         │ 4. Redirect do TPay   │                       │
         │<──────────────────────│                       │
         │                       │                       │
         │ 5. Autoryzacja BLIK   │                       │
         │──────────────────────────────────────────────>│
         │                       │                       │
         │                       │ 6. Webhook ALIAS_REGISTER
         │                       │<──────────────────────│
         │                       │                       │
         │ 7. Potwierdzenie      │                       │
         │<──────────────────────│                       │
         │                       │                       │
```

### 2.2 Płatności Cykliczne (Automatyczne)

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Cron Job      │     │   Edge Function │     │   TPay API      │
│   (7:00 daily)  │     │                 │     │                 │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         │ 1. Trigger            │                       │
         │──────────────────────>│                       │
         │                       │                       │
         │                       │ 2. Pobierz subskrypcje
         │                       │    do odnowienia      │
         │                       │                       │
         │                       │ 3. POST /transactions │
         │                       │ (recursive + alias)   │
         │                       │──────────────────────>│
         │                       │                       │
         │                       │ 4. Wynik transakcji   │
         │                       │<──────────────────────│
         │                       │                       │
         │                       │ 5. Aktualizacja DB    │
         │                       │    + utworzenie       │
         │                       │    zamówienia         │
         │                       │                       │
```

---

## 3. Endpointy API

### 3.1 Tworzenie Subskrypcji (Pierwsza Płatność)

**Endpoint:** `POST /functions/v1/create-tpay-subscription`

**Request Body:**
```json
{
  "items": [
    {
      "product_id": "uuid",
      "product_name": "GO NATIVE Karma dla Dorosłych Psów",
      "product_image": "https://...",
      "price": 189.00,
      "quantity": 1,
      "size": "12kg"
    }
  ],
  "frequency": "4_weeks",
  "shipping_name": "Jan Kowalski",
  "shipping_street": "ul. Przykładowa 1",
  "shipping_city": "Warszawa",
  "shipping_postal_code": "00-001"
}
```

**Response:**
```json
{
  "success": true,
  "subscription_id": "uuid",
  "redirect_url": "https://secure.tpay.com/..."
}
```

### 3.2 Webhook Powiadomień

**Endpoint:** `POST /functions/v1/tpay-webhook`

**URL Webhook:** `https://mpxkmkfxhfgapreervjw.supabase.co/functions/v1/tpay-webhook`

**Obsługiwane Powiadomienia:**

| Typ | Opis |
|-----|------|
| `ALIAS_REGISTER` | Rejestracja aliasu BLIK - aktywacja subskrypcji |
| `correct` / `TRUE` | Płatność zakończona sukcesem |
| `failed` | Płatność nieudana |

**Przykład Powiadomienia ALIAS_REGISTER:**
```json
{
  "aliasType": "PAYID",
  "aliasValue": "alias_unique_id",
  "aliasStatus": "registered",
  "transactionId": "TR-XXX-XXXX"
}
```

### 3.3 Pobieranie Płatności Cyklicznych

**Endpoint:** `POST /functions/v1/charge-tpay-subscription`

**Trigger:** Cron job codziennie o 7:00

**Logika:**
1. Pobierz aktywne subskrypcje gdzie `next_delivery_date <= NOW() + 2 days`
2. Dla każdej subskrypcji wykonaj transakcję z aliasem
3. Zapisz wynik w `subscription_payment_attempts`
4. Przy sukcesie: utwórz zamówienie, zaktualizuj `next_delivery_date`
5. Przy błędzie: zwiększ `failed_payment_count`, wyślij powiadomienie

---

## 4. Struktura Bazy Danych

### 4.1 Tabela: subscriptions

```sql
CREATE TABLE subscriptions (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL,
  status VARCHAR DEFAULT 'pending',
  delivery_frequency VARCHAR DEFAULT '4_weeks',
  next_delivery_date TIMESTAMP NOT NULL,
  total_price DECIMAL NOT NULL,
  
  -- TPay BLIK Recurring
  payment_provider VARCHAR DEFAULT 'tpay',
  tpay_blik_alias VARCHAR,
  tpay_alias_status VARCHAR,
  
  -- Płatności
  is_first_order BOOLEAN DEFAULT true,
  order_count INTEGER DEFAULT 0,
  failed_payment_count INTEGER DEFAULT 0,
  last_payment_date TIMESTAMP,
  
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

### 4.2 Tabela: subscription_items

```sql
CREATE TABLE subscription_items (
  id UUID PRIMARY KEY,
  subscription_id UUID REFERENCES subscriptions(id),
  product_id UUID,
  product_name VARCHAR NOT NULL,
  product_image VARCHAR NOT NULL,
  price DECIMAL NOT NULL,
  quantity INTEGER DEFAULT 1,
  size VARCHAR,
  created_at TIMESTAMP DEFAULT NOW()
);
```

### 4.3 Tabela: subscription_payment_attempts

```sql
CREATE TABLE subscription_payment_attempts (
  id UUID PRIMARY KEY,
  stripe_subscription_id VARCHAR, -- używane też dla TPay
  user_id UUID,
  customer_email VARCHAR,
  amount_due DECIMAL NOT NULL,
  status VARCHAR DEFAULT 'pending',
  attempt_number INTEGER DEFAULT 1,
  failure_reason VARCHAR,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

---

## 5. Parametry Transakcji TPay

### 5.1 Pierwsza Transakcja (Rejestracja)

```json
{
  "amount": 170.10,
  "description": "GO NATIVE Subskrypcja - Pierwsze zamówienie",
  "hiddenDescription": "subscription_uuid",
  "payer": {
    "email": "klient@example.com",
    "name": "Jan Kowalski"
  },
  "pay": {
    "groupId": 150,
    "recursive": {
      "type": "FIRST",
      "expirationDate": "2027-01-15"
    }
  },
  "callbacks": {
    "payerUrls": {
      "success": "https://gonative.pl/subscriptions?status=success",
      "error": "https://gonative.pl/subscriptions?status=error"
    },
    "notification": {
      "url": "https://mpxkmkfxhfgapreervjw.supabase.co/functions/v1/tpay-webhook"
    }
  }
}
```

### 5.2 Transakcja Cykliczna

```json
{
  "amount": 179.55,
  "description": "GO NATIVE Subskrypcja - Odnowienie #2",
  "hiddenDescription": "recurring_subscription_uuid",
  "payer": {
    "email": "klient@example.com",
    "name": "Jan Kowalski"
  },
  "pay": {
    "groupId": 150,
    "recursive": {
      "type": "SUBSEQUENT",
      "alias": "alias_value_from_registration"
    }
  },
  "callbacks": {
    "notification": {
      "url": "https://mpxkmkfxhfgapreervjw.supabase.co/functions/v1/tpay-webhook"
    }
  }
}
```

---

## 6. Polityka Rabatów

| Zamówienie | Rabat | Opis |
|------------|-------|------|
| Pierwsze | 10% | Rabat powitalny dla nowych subskrybentów |
| Kolejne | 5% | Stały rabat za lojalność |

---

## 7. Obsługa Błędów Płatności

### 7.1 Strategia Retry

1. **Pierwsza próba** - dzień planowanej dostawy minus 2 dni
2. **Próby kolejne** - codziennie przez 3 dni
3. **Po 3 nieudanych próbach** - subskrypcja zostaje wstrzymana (status: `paused`)

### 7.2 Powiadomienia

- Email do klienta po każdej nieudanej próbie
- Email do admina przy wstrzymaniu subskrypcji
- Przypomnienie SMS (planowane)

---

## 8. Bezpieczeństwo

### 8.1 Przechowywanie Danych

- Aliasy BLIK przechowywane w bazie danych Supabase
- Brak przechowywania danych karty/konta bankowego
- Szyfrowanie danych w tranzycie (HTTPS)
- Row Level Security (RLS) na wszystkich tabelach

### 8.2 Weryfikacja Webhooków

- Weryfikacja adresu IP TPay (opcjonalnie)
- Sprawdzanie struktury payloadu
- Logowanie wszystkich zdarzeń

---

## 9. Adresy URL

### 9.1 Produkcja

| Zasób | URL |
|-------|-----|
| Strona główna | https://gonative.pl |
| Strona subskrypcji | https://gonative.pl/abonament |
| Checkout | https://gonative.pl/checkout |
| Panel klienta | https://gonative.pl/subscriptions |
| Webhook TPay | https://mpxkmkfxhfgapreervjw.supabase.co/functions/v1/tpay-webhook |

### 9.2 Testowanie

| Zasób | URL |
|-------|-----|
| Preview | https://id-preview--8c489f35-a339-4516-81bd-6026c72c48a0.lovable.app |
| Alternatywny | https://ta0kmoq2t.lovable.app |

---

## 10. Kontakt Techniczny

**Developer:** Lovable.dev  
**Email:** kontakt@gonative.pl  
**Telefon:** +48 XXX XXX XXX

---

## 11. Załączniki

### Do dołączenia:
1. Screenshot strony produktu subskrypcyjnego
2. Screenshot procesu checkout z wyborem TPay
3. Screenshot strony przekierowania TPay
4. Screenshot panelu zarządzania subskrypcją klienta
5. Video demo całego procesu (opcjonalnie)

---

*Dokument wygenerowany: 15.01.2026*  
*Wersja: 1.0*
