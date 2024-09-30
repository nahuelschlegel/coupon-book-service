```typescript
// API Service Code
import db from PostgreSQL
import awsSQS from AWS

async function main({ coupon_book_id, code_pattern, quantity, codes }) {
  couponBook = db.coupon_books.find({ where: { id: coupon_book_id } })

  if (couponBook.length == 0)
    throw Error(400, "Invalid input.")

  if (codes.length) {
    couponAlreadyExist = db.coupons.find({ where: { coupon_book_id: coupon_book_id, code: codes } })

    if (couponAlreadyExist)
      throw Error(422, "Validation error.")

    codesToInsert = codes.map((val) => ({
      code: val,
      coupon_book_id: coupon_book_id
    }))

    db.coupons.insertMany(codesToInsert)
  } else if (code_pattern && quantity) {
    awsSQS.enqueueMessage("generateCodes", { coupon_book_id, code_pattern, quantity })
  } else {
    throw Error(400, "Invalid input.")
  }

  return { code: 202 }
}

main();
```

```typescript
// Worker Service Code
import db from PostgreSQL
import awsSQS from AWS

async function fetchExistingCoupons(pattern: string) {
  const patternRegex = pattern
    .replace(/#/g, '\\d')
    .replace(/\?/g, '[A-Z]');

  try {
    const result = await db.query(`SELECT code FROM coupons WHERE code ~ ^${patternRegex}$`);
    return result.map(row => row.code);
  } catch (error) {
    throw Error(422, 'Error fetching existing coupons.');
  }
}

function getRandomNumber(max: number) {
  return Math.round(Math.random() * max);
}

function generateCode(pattern: string) {
  letters = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
  numbers = "0123456789"

  return pattern.replace(/[#\?]/g, (match) => {
    if (match === '#') return numbers[getRandomNumber(numbers.length)];
    if (match === '?') return letters[getRandomNumber(letters.length)];
    return match;
  });
}

async function generateCouponCodes({
  pattern,
  quantity,
}) {
  maxAttempts = quantity * 3
  existingCoupons = await fetchExistingCoupons(pattern);
  coupons = new Set(existingCoupons);

  attempts = 0;
  while (coupons.size < quantity + existingCoupons.length) {
    if (attempts >= maxAttempts) {
      throw Error(422, `Failed to generate ${quantity} unique coupons after ${maxAttempts} attempts.`);
    }

    code = generateCode(pattern);
    if (!coupons.has(code)) {
      coupons.add(code);
    }
    attempts++;
  }

  return Array.from(coupons).filter(existingCoupons);
}

@OnMessageReceived('generateCodes')
async function generateCodes(message, { coupon_book_id, code_pattern, quantity }) {
  try {
    await generateCouponCodes({ coupon_book_id, code_pattern, quantity });
    message.acknowledge()
  } catch(error) {
    log("error generating all codes, don't ack and lets try again in 1 minute")
  }
}
```
