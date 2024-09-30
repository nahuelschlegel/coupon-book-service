```typescript
// API Service Code
import db from PostgreSQL
import redis from Redis

async function lockCoupon(couponAssignment) {
  try {
    result = redis("SET coupon-lockForRedemption-{couponAssigmentId} NX EX 120")

    if (!result)
      throw Error(409, "Conflict.", "The coupon is already locked.")

    return "Coupon temporarily locked, you have 2 minutes to finish redemption.";
  } catch (error) {
    throw Error(422, 'Error locking coupon');
  }
}

async function main({ user_id, coupon_id }) {
  if (!user_id || !coupon_id)
    throw Error(400, "Invalid input.")

  try {
    db("BEGIN")
    couponAssignment = db.couponAssignments.find({ where: { user_id: user_id, coupon_id: coupon_id } }).forUpdate()
    if (!couponAssignment)
      throw Error(400, "Invalid input.")

    coupon = db.coupon.find({ where: { id: coupon_id } })
    if (!coupon)
      throw Error(400, "Invalid input.")

    couponBook = db.couponBook.find({ where: { id: coupon.coupon_book_id } })
    if (!couponBook || couponBook.max_per_user <= couponAssignment.redemption_count)
      throw Error(400, "Invalid input.")

    result = await lockCoupon(couponAssignment)

    return result
  } catch (error) {
    return error
  } finally {
    db("ROLLBACK")
    db("END")
  }
}

main();
```
