# `POST /coupons`:

### Endpoint logic:

This endpoint validates that the title is not empty and then generates a CouponBook in database with the received data or the default values.

Only Admins can create CouponBooks. The access token received must be from an admin role.

### Request format:

```JSON
{
  "title": "Premium",
  "description": "Coupon book description",
  "max_per_user": 10,
  "allow_multiple_redemptions": false
}
```

### Response format:

Code: `200`

```JSON
{
	"id": 1,
	"title": "Premium",
	"description": "Coupon book description",
	"max_per_user": 10,
	"allow_multiple_redemptions": false,
	"created_at": "1970-01-01T00:00:00.000Z",
	"updated_at": "1970-01-01T00:00:00.000Z"
}
```

Error codes: `400`, `401`, `422`.

# `POST /coupons/codes`:

### Endpoint logic:

This endpoint is in charge of the creation of Coupons in the coupons table associating those coupons to a CouponBook.
Checks that the coupon book id exists on the CouponBooks table;
- If a list of codes is provided, it creates the codes on the Coupons table associated to the given Coupon Book.
- If not, checks if a code_pattern and a quantity was provided, if it is OK it proceeds to create the coupons.
	- This **creates a message in the AWS SQS** to asynchronously generate the codes via the Worker Service.
- If no codes or pair of code_pattern and quantity was provided, returns a 422 error.

Only Admins can create Coupon Codes. The access token received must be from an admin role.

[Generation Logic Pseudocode](./generationLogic.md)

### Request format:

#### Required fields:
- `coupon_book_id`
- one of `codes` or `code_pattern and quantity`

1. **_Option One:_** First option JSON with code pattern and the amount of codes to be auto generated:

```JSON
{
  "coupon_book_id": 1,
  "code_pattern": "COUPON-######-??????", // String code where each '#' represents a number and each '?' represents an uppercase letter.
  "quantity": 15,
}
```

2. **_Option Two:_** A comma separated list of code values:

```JSON
{
  "coupon_book_id": 1,
  "codes": [ "CODE10017", "25OFF", "HOTSALE15", "HOTSALE100USD" ]
}
```

### Response format:

Code: `202`

Description: Validations passed. Message queued to generate codes in Coupon Book, based on list code or code pattern.

**Empty response**

```
```

Error codes: `400`, `401`, `422`.

# `POST /coupons/assign`:

### Endpoint logic:

This endpoint validates coupon_book_id and user_id are not empty and are valid ids.

Checks the number of already assigned coupons for the given book ID and user ID, and validates against the max_per_user parameter on the Coupon Book.

Finds available coupons based on coupons of the given book ID and that doesn't have any assignment for the given user ID.

Finally picks a random code from the available coupons for this user and creates a CouponAssignment on the DB.

[Random code assignment Pseudocode](./randomCodeAssignment.md)

#### Required fields:
- `user_id`
- `coupon_book_id`

### Request format:

```JSON
{
	"user_id": 15,
	"coupon_book_id": 2
}
```

### Response format:

Code: `200`

```JSON
{
	"id": 1,
	"coupon_id": 5,
	"user_id": 15,
	"redemption_count": 0,
	"max_redemptions": 1,
	"created_at": "1970-01-01T00:00:00.000Z",
	"updated_at": "1970-01-01T00:00:00.000Z"
}
```

Error codes: `400`, `401`, `422`.

# `POST /coupons/assign/{code}`:

### Endpoint logic:

This endpoint validates user_id and code are valid ids.

Checks the number of already assigned coupons for the given coupon's book ID, and the given user ID, and validates against the max_per_user parameter on the Coupon Book.

Finally, if everything it's OK, creates a CouponAssignment on the DB.

#### Required fields:
- `user_id`

### Request format:

```JSON
{
	"user_id": 15
}
```

### Response format:

Code: `200`

```JSON
{
	"id": 1,
	"coupon_id": 2,
	"user_id": 15,
	"redemption_count": 0,
	"max_redemptions": 1,
	"created_at": "1970-01-01T00:00:00.000Z",
	"updated_at": "1970-01-01T00:00:00.000Z"
}
```

Error codes: `400`, `401`, `422`.

# `POST /coupons/lock/{code}`:

### Endpoint logic:

This endpoint uses Redis to temporarily lock the coupon code, before making the needed validations.

Uses Redis `SET coupon-lockForRedemption-{couponAssigmentId} NX EX 120`, this creates a registry in Redis that auto-expires after 120 seconds, it has the format `coupon-lockForRedemption-{couponAssigmentId}`, so it only affects that user's coupon. If the user tries to redeem it from another device at the same time, as the registry already exists the lock creation will fail and reject the locking process.

This kind of locking process prevents race conditions and ensure data integrity, and is faster than SQL row level locking.

#### Required fields:
- `code`
- `user_id`

### Request format:

```JSON
{
	"user_id": 15
}
```

### Response format:

Code: `200`

```
```

Error codes: `400`, `401`, `409`, `422`.

# `POST /coupons/unlock/{code}`:

### Endpoint logic:

This endpoint uses Redis to unlock the coupon code, before making the needed validations.

Finds the CouponAssignment and then checks if exists a lock in Redis using the same format key used for locking. If it's found it calls to Redis to DELETE the registry, unlocking the code for the user.

#### Required fields:
- `code`
- `user_id`

### Request format:

```JSON
{
	"user_id": 15
}
```

### Response format:

Code: `200`

```
```

Error codes: `400`, `401`, `409`, `422`.

# `POST /coupons/redeem/{code}`:

### Endpoint logic:

This endpoint uses SQL row level locking to temporarily lock the coupon code during redemption, before making the needed validations. If the user tries to redeem it from another device at the same time, as the row is locked it cannot be read and the operation will fail.

The SELECT ... FOR UPDATE, locks the row affected by the query until the transaction ends.

If every check goes well, updates the CouponsAssignment and creates a new registry in the CouponRedemptions.

This kind of locking process prevents race conditions and ensure data integrity.

#### Required fields:
- `code`
- `user_id`

### Request format:

```JSON
{
	"user_id": 15
}
```

### Response format:

Code: `200`

```JSON
{
	"id": 1,
	"coupon_assignment_id": 1,
	"status": "successful",
	"created_at": "1970-01-01T00:00:00.000Z"
}
```

Error codes: `400`, `401`, `409`, `422`.

# `GET /coupons/user/{id}`:

#### Required fields:
- `id`: user ID param

### Response format:

Code: `200`

```JSON
[
	{
		"id": 1,
		"coupon_id": 2,
		"user_id": 3,
		"redemption_count": 0,
		"max_redemptions": 1,
		"created_at": "1970-01-01T00:00:00.000Z",
		"updated_at": "1970-01-01T00:00:00.000Z"
	},
	{
		"id": 2,
		"coupon_id": 3,
		"user_id": 3,
		"redemption_count": 1,
		"max_redemptions": 1,
		"last_redeemed_at": "1970-01-01T00:00:00.000Z",
		"created_at": "1970-01-01T00:00:00.000Z",
		"updated_at": "1970-01-01T00:00:00.000Z"
	}
]
```

Error codes: `400`, `401`, `422`.

## Error codes

| Code | Message                            |
| :--: | ---------------------------------- |
| 400  | Invalid input                      |
| 401  | Access token is missing or invalid |
| 422  | Validation exception               |
