নিশ্চয়ই — আমি আপনার `memberships` টেবিলটিকে **পূর্ণাঙ্গ, প্রফেশনাল এবং জিম ক্লায়েন্ট ম্যানেজমেন্টের জন্য ফিচার-রিচ** করে সাজিয়ে দিচ্ছি, যেখানে বিলিং, সাবস্ক্রিপশন টাইপ, পেমেন্ট, স্ট্যাটাস, রিনিউয়াল, নোট এবং অডিট ট্র্যাকিং সব কভার থাকবে।

```sql
-- টেবিল: memberships
-- উদ্দেশ্য: প্রতিটি ক্লায়েন্টের মেম্বারশিপ প্ল্যান, ফি, স্ট্যাটাস, রিনিউয়াল এবং বেনিফিটের তথ্য সংরক্ষণ
CREATE TABLE memberships (
    membership_id BIGINT AUTO_INCREMENT PRIMARY KEY, -- ইউনিক মেম্বারশিপ আইডি
    client_id BIGINT NOT NULL, -- কোন ক্লায়েন্টের মেম্বারশিপ (clients টেবিল রেফারেন্স)
    
    plan_name VARCHAR(100) NOT NULL, -- প্ল্যানের নাম (Basic, Premium, VIP ইত্যাদি)
    plan_type ENUM('monthly', 'quarterly', 'half-yearly', 'yearly', 'custom') NOT NULL DEFAULT 'monthly', -- সাবস্ক্রিপশন টাইপ
    start_date DATE NOT NULL, -- মেম্বারশিপ শুরুর তারিখ
    end_date DATE NOT NULL, -- মেম্বারশিপ শেষ হওয়ার তারিখ
    renewal_date DATE DEFAULT NULL, -- পরবর্তী রিনিউয়াল তারিখ
    
    monthly_fee DECIMAL(10, 2) DEFAULT 0.00, -- মাসিক ফি
    trainer_fee DECIMAL(10, 2) DEFAULT 0.00, -- ট্রেইনার ফি
    supplement_fee DECIMAL(10, 2) DEFAULT 0.00, -- সাপ্লিমেন্ট বা প্রোডাক্ট ফি
    discount_amount DECIMAL(10, 2) DEFAULT 0.00, -- ডিসকাউন্ট পরিমাণ
    
    -- মোট টাকা ক্যালকুলেশন
    total_amount DECIMAL(10, 2) GENERATED ALWAYS AS (
        (monthly_fee + trainer_fee + supplement_fee) - discount_amount
    ) STORED, -- ফাইনাল পরিশোধযোগ্য পরিমাণ
    
    status ENUM('active', 'inactive', 'paused', 'expired') DEFAULT 'active', -- মেম্বারশিপ স্ট্যাটাস
    auto_renew BOOLEAN DEFAULT FALSE, -- অটো রিনিউয়াল সক্রিয় কিনা
    notes TEXT DEFAULT NULL, -- বিশেষ নোট বা বেনিফিটের বিবরণ
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP, -- রেকর্ড তৈরি হওয়ার সময়
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP, -- সর্বশেষ আপডেটের সময়
    
    -- ডেটা ভ্যালিডেশন
    CONSTRAINT chk_fees_non_negative CHECK (
        monthly_fee >= 0
        AND trainer_fee >= 0
        AND supplement_fee >= 0
        AND discount_amount >= 0
    ),
    CONSTRAINT chk_valid_dates CHECK (end_date >= start_date),
    
    -- ফরেন কি: ক্লায়েন্ট ডিলিট হলে এই মেম্বারশিপও ডিলিট হবে
    FOREIGN KEY (client_id) REFERENCES clients (client_id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- পারফরম্যান্স ইনডেক্স
CREATE INDEX idx_memberships_client_id ON memberships (client_id);
CREATE INDEX idx_memberships_status ON memberships (status);
CREATE INDEX idx_memberships_dates ON memberships (start_date, end_date);

-- উদাহরণ ডেটা ইনসার্ট
INSERT INTO memberships (
    client_id, plan_name, plan_type, start_date, end_date, renewal_date,
    monthly_fee, trainer_fee, supplement_fee, discount_amount,
    status, auto_renew, notes
) VALUES (
    101, 'Premium Plan', 'monthly', '2025-08-01', '2025-08-31', '2025-09-01',
    2000.00, 500.00, 800.00, 300.00,
    'active', TRUE, 'Includes personal trainer & weekend yoga sessions'
);

-- উদাহরণ কুয়েরি
-- 1) সক্রিয় মেম্বারশিপের তালিকা
SELECT * FROM memberships WHERE status = 'active' ORDER BY end_date ASC;

-- 2) নির্দিষ্ট ক্লায়েন্টের সব মেম্বারশিপ
SELECT * FROM memberships WHERE client_id = 101 ORDER BY start_date DESC;

-- 3) মেয়াদ উত্তীর্ণ মেম্বারশিপ
SELECT * FROM memberships WHERE end_date < CURDATE() AND status != 'expired';

-- 4) মোট আয় হিসাব (ফাইনাল পরিমাণ)
SELECT SUM(total_amount) AS total_revenue FROM memberships WHERE status = 'active';
```

💡 **অতিরিক্ত ফিচার আইডিয়া:**

1. `pause_start` এবং `pause_end` কলাম — মেম্বারশিপ হোল্ড/ফ্রিজ করার জন্য।
2. `membership_type_id` — ভিন্ন ধরনের প্ল্যানের জন্য আলাদা টেবিল (যেমন group class, personal training, combo)।
3. `payment_status` ENUM কলাম — প্রতিটি মেম্বারশিপের পেমেন্ট ট্র্যাকিং।
4. `invoice_url` — প্রতিটি মেম্বারশিপের PDF ইনভয়েস সংরক্ষণ।

আপনি চাইলে আমি এখনই এই **memberships + billing\_info + subscriptions + clients** সব টেবিলের **পুরো ERD + JOIN কুয়েরি উদাহরণ** বানিয়ে দিতে পারি, যাতে জিম ম্যানেজমেন্ট সিস্টেম পুরোপুরি রিলেশনাল ও কভারড হয়।

আপনি কি সেটা দেখতে চান?
