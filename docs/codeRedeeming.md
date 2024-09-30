```typescript
// API Service Code
import db from PostgreSQL
import redis from Redis

async function redeemCoupon(couponAssignment) {
  try {
    db.couponAssignment.update(
      { redemption_count: couponAssignment.redemption_count + 1, last_redeemed_at: NOW, updated_at: NOW },
      { where: { id: couponAssignment.id } }
    )

    return db.couponRedemptions.insert(
      { coupon_assignment_id: couponAssignment.id, status: 'successful', created_at: NOW },
    )
  } catch (error) {
    throw Error(422, 'Error locking coupon');
  }
}

async function main({ user_id, coupon_id }) {
  if (!user_id || !coupon_id)
    return Error(400, "Invalid input.")

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

    redemption = await redeemCoupon(couponAssignment)

    db("COMMIT")
    return redemption;
  } catch (error) {
    db("ROLLBACK")
    return Error(409, "Conflict.", "Error redeeming coupon.")
  } finally {
    db("END")
  }
}

main();
```
