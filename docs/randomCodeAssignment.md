```typescript
// API Service Code
import db from PostgreSQL
import awsSQS from AWS

function getRandomNumber(max: number) {
  return Math.round(Math.random() * max);
}

async function getAvailableCoupons(userId: number, couponBookId: number): Promise<number[]> {
  try {
    const result = await db.find("Find coupons from this coupon_book_id not assigned to the user yet, based on coupons and couponAssignments");
    return result.map(row => row.id);
  } catch (error) {
    throw Error(422, 'Error fetching available coupons.');
  }
}

async function assignRandomCoupon({ user_id, coupon_book_id }) {
  try {
    couponBook = db.couponBooks.find({ where: { id: coupon_book_id } })
    user = db.users.find({ where: { id: user_id } })

    if (!couponBook || !user)
      throw Error(400, "Invalid input.")

    hasReachedMax = db.find("Count of already assigned coupons versus max_per_user from coupon book")

    if (hasReachedMax)
      throw Error(422, "Validation error.", "Max amount of codes per book reached.")

    const availableCoupons = await getAvailableCoupons(userId, couponBookId);

    if (availableCoupons.length === 0) {
      return Error(422, "No coupons available for this user.")
    }

    const selectedCouponId = availableCoupons[getRandomNumber(0, availableCoupons.length)];

    return await client.insert({
      coupon_id: selectedCouponId,
      user_id: user_id,
      redemption_count: 0,
      max_redemptions: couponBook.allow_multiple_redemptions ? couponBook.max_per_user : 1,
    });
  } catch (error) {
    throw Error(422, 'Error assigning coupon');
  }
}

async function main({ user_id, coupon_book_id }) {
  if (!user_id || !coupon_book_id)
    throw Error(400, "Invalid input.")

  return await assignRandomCoupon(codesToInsert)
}

main();
```
