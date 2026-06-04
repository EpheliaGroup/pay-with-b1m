# B1M Payment SDK

Drop-in payment widget for merchants integrating with the B1M payment platform. Supports payments via the B1M app, card registration with Stripe, and direct card payments.

## Installation

### CDN (recommended)

```html
<script src="https://unpkg.com/@epheliagroup/pay-with-b1m/dist/pay-with-b1m.dist.js"></script>
<link href="https://unpkg.com/@epheliagroup/pay-with-b1m/dist/pay-with-b1m.dist.css" rel="stylesheet" />
```

### npm

```bash
npm install @epheliagroup/pay-with-b1m
```

## Environments

| Parameter | Sandbox (UAT) | Production |
|-----------|---------------|------------|
| API Base URL | `https://gateway.b1m.uat.ephelia.dev` | TBD |
| Auth URL | `https://auth.b1m.uat.ephelia.dev` | TBD |
| Payment Page | `https://pay.b1m.uat.ephelia.dev` | TBD |
| Realm | `b1m` | `b1m` |
| Client ID | `b1m-client` | `b1m-client` |

## Quick Start — Redirect Integration

The simplest integration: create a payment session on your server and redirect the customer to the hosted payment page.

### Step 1: Authenticate (server-side)

```javascript
const tokenResponse = await fetch(
  'https://gateway.b1m.uat.ephelia.dev/auth/realms/b1m/protocol/openid-connect/token',
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'password',
      client_id: 'b1m-client',
      username: 'YOUR_MERCHANT_USERNAME',
      password: 'YOUR_MERCHANT_PASSWORD',
    }),
  }
);
const { access_token } = await tokenResponse.json();
```

### Step 2: Create a payment session (server-side)

```javascript
const response = await fetch('https://gateway.b1m.uat.ephelia.dev/acquiring/v1/payment', {
  method: 'POST',
  headers: {
    Authorization: `Bearer ${access_token}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    amount: '10.00',
    currency: 'EUR',
    reason: 'Order #1234',
    successUrl: 'https://pay.b1m.uat.ephelia.dev/success.html',
    errorUrl: 'https://pay.b1m.uat.ephelia.dev/error.html',
    providers: ['B1M', 'CARD_REGISTRATION'],
  }),
});
const { sessionId } = await response.json();
```

### Step 3: Redirect the customer

```javascript
window.location.href = 'https://pay.b1m.uat.ephelia.dev/?orderId=' + sessionId;
```

The customer completes the payment on the hosted page and is redirected to your `successUrl` or `errorUrl`.

## Available Providers

| Provider | Description |
|----------|-------------|
| `B1M` | Payment via the B1M mobile app (QR code on desktop, deep link on mobile) |
| `CARD_REGISTRATION` | Card payment with Stripe + automatic customer registration |
| `STRIPE` | Card payment with Stripe (no registration) |

Include one or more in the `providers` array when creating a payment session.

## Embedded Widget Integration

For a fully customized experience, embed the payment widget directly in your page.

### HTML Setup

```html
<div id="payment-container"></div>

<script src="https://unpkg.com/@epheliagroup/pay-with-b1m/dist/pay-with-b1m.dist.js"></script>
<link href="https://unpkg.com/@epheliagroup/pay-with-b1m/dist/pay-with-b1m.dist.css" rel="stylesheet" />
```

### Initialize the Widget

```javascript
PayWithB1M.render(document.querySelector('#payment-container'), {
  // Required: payment lifecycle callbacks (implement these on your server)
  createPaymentSession,
  fetchPaymentDataBySessionId,
  initiatePayment,
  updatePayment,
  approveCardPayment,

  // Optional: streaming payments
  createStreamPaymentIntent,
  approveStreamPayment,
  rejectStreamPayment,
  fetchStreamPaymentIntent,

  // Optional: card registration flow
  quickRegisterCustomer,

  // Callbacks
  successPaymentCallback: (data) => { /* handle success */ },
  failurePaymentCallback: (err) => { /* handle failure */ },
  onModalClose: (error) => { if (error) console.error(error); },

  // Config
  baseUrl: 'https://gateway.b1m.uat.ephelia.dev',
  renderPayWithToonieButton: true,
  renderStreamWithToonieButton: false,
  renderPayWithCardButton: true,
});
```

## Callback Reference

Each callback function wraps a call to the acquiring API. Below are the expected signatures and return values.

### `createPaymentSession({ amount, currency, reason, providers })`

Creates a new payment session.

```javascript
const createPaymentSession = async ({ amount, currency, reason, providers }) => {
  const res = await fetch(`${baseUrl}/acquiring/v1/payment`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ amount, currency, reason, successUrl, errorUrl, providers }),
  });
  const data = await res.json();
  return {
    paymentSessionId: data.sessionId,
    status: data.status,
    amount: data.amount,
    currency: data.currency,
    successUrl: data.successUrl,
    errorUrl: data.errorUrl,
    reason: data.reason,
    merchantDisplayName: data.displayName,
    providers: data.selectedProviders,
  };
};
```

### `fetchPaymentDataBySessionId(paymentSessionId)`

Retrieves payment session details.

```javascript
const fetchPaymentDataBySessionId = async (paymentSessionId) => {
  const res = await fetch(`${baseUrl}/acquiring/v1/payment/paymentSession/${paymentSessionId}`, {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  });
  const data = await res.json();
  if (res.status === 422) return { error: data.id };
  return {
    paymentSessionId: data.sessionId,
    status: data.status,
    amount: data.amount,
    currency: data.currency,
    reason: data.reason,
    successUrl: data.successUrl,
    errorUrl: data.errorUrl,
    merchantDisplayName: data.displayName,
    providers: data.selectedProviders,
    createdAt: data.created,
    lastUpdated: data.last_updated,
    merchantAddress: {
      addressLine1: data.address.addressLine1,
      addressLine2: data.address.addressLine2,
      city: data.address.city,
      country: data.address.country,
      postalCode: data.address.postalCode,
      state: data.address.state,
    },
  };
};
```

### `initiatePayment({ paymentSessionId, provider })`

Initiates payment with the selected provider. Returns provider-specific data.

```javascript
const initiatePayment = async ({ paymentSessionId, provider }) => {
  const res = await fetch(`${baseUrl}/acquiring/v1/payment/initiate`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ paymentSessionId, provider }),
  });
  const data = await res.json();
  if (!res.ok) return { error: data.message || data.error || 'Unknown error', status: data.status || res.status };
  return {
    amount: data.amount,
    currency: data.currency,
    reason: data.reason,
    status: data.status,
    clientSecret: data.clientSecret,
    stripePaymentIntentId: data.paymentIntentId,
    feeAmount: data.feeAmount,
    otp: data.provider.otp,
    offersSessionId: data.provider.paymentOfferSessionId,
    paymentShortReference: data.provider.shortReference,
  };
};
```

### `updatePayment(paymentSessionId, paymentStatus)`

Updates the payment status (e.g., `APPROVED`, `REJECTED`).

```javascript
const updatePayment = async (paymentSessionId, paymentStatus) => {
  return await fetch(`${baseUrl}/acquiring/v1/payment/${paymentSessionId}/status/${paymentStatus}`, {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
  });
};
```

### `approveCardPayment(paymentSessionId)`

Confirms a Stripe card payment after `stripe.confirmCardPayment` succeeds.

```javascript
const approveCardPayment = async (paymentSessionId) => {
  return await fetch(`${baseUrl}/acquiring/v1/payment/${paymentSessionId}/approve`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
  });
};
```

### `quickRegisterCustomer({ email, currency, firstName, lastName, prefix, mobileNumber, birthDate, paymentSessionId })`

Registers a new customer during the card registration payment flow.

```javascript
const quickRegisterCustomer = async ({ email, currency, firstName, lastName, prefix, mobileNumber, birthDate, paymentSessionId }) => {
  const res = await fetch(`${baseUrl}/acquiring/v1/payment/quickRegistration`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, currency, firstName, lastName, prefix, mobileNumber, birthDate, paymentSessionId }),
  });
  const data = await res.json();
  return { walletId: data.walletId, partyId: data.partyId };
};
```

## Error Handling

The acquiring API returns validation errors with HTTP status 422 in this format:

```json
{
  "code": "ValidationError",
  "id": "FEES_NOT_RETRIEVED",
  "message": "Descriptive error message",
  "errors": null
}
```

Always check `data.message` (not `data.error`) when handling non-OK responses from the API.

## Payment Flow Diagrams

### B1M App Payment

```
Merchant server                      Customer browser                    B1M App
     |                                     |                               |
     | 1. POST /acquiring/v1/payment       |                               |
     |------------------------------------>|                               |
     | 2. Redirect to ?orderId=sessionId   |                               |
     |------------------------------------>|                               |
     |                                     | 3. Shows QR code / deep link  |
     |                                     |------------------------------>|
     |                                     |                               | 4. User approves
     |                                     | 5. Redirect to successUrl     |
     |                                     |<------------------------------|
```

### Card Registration Payment

```
Customer browser                     Acquiring API                    Stripe
     |                                     |                            |
     | 1. Fill registration form           |                            |
     | 2. POST /quickRegistration          |                            |
     |------------------------------------>|                            |
     | 3. POST /initiate (CARD_REG)        |                            |
     |------------------------------------>| 4. Create PaymentIntent    |
     |                                     |--------------------------->|
     | 5. Receive clientSecret             |                            |
     |<------------------------------------|                            |
     | 6. stripe.confirmCardPayment()      |                            |
     |-------------------------------------------------------->------->|
     | 7. POST /approve                    |                            |
     |------------------------------------>| 8. Confirm + credit wallet |
     |                                     |--------------------------->|
     | 9. Redirect to successUrl           |                            |
```

## Notes

- Minimum payment amount for card payments is **EUR 0.50** (Stripe requirement).
- The `CARD_REGISTRATION` provider creates a B1M account for the customer during payment.
- Payment sessions are single-use — do not resubmit the registration form for the same session.

## License

MIT
