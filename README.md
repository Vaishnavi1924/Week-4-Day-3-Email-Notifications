# Week-4-Day-3-Email-Notifications
Customer places order         ↓ Order confirmation email  Vendor receives order         ↓ New order notification  Order status changed         ↓ Customer receives update

1. Add email settings to .env

Open:

backend/.env

Add:

SMTP_SERVICE=Gmail
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password
EMAIL_FROM=your_email@gmail.com

2. Create email configuration

Create:

backend/config/email.js

Add:

const nodemailer = require("nodemailer");

const transporter = nodemailer.createTransport({
  service: process.env.SMTP_SERVICE,
  auth: {
    user: process.env.SMTP_USER,
    pass: process.env.SMTP_PASS
  }
});

const verifyEmailConnection = async () => {
  try {
    await transporter.verify();
    console.log("Email server is ready");
  } catch (error) {
    console.error(
      "Email connection error:",
      error.message
    );
  }
};

module.exports = {
  transporter,
  verifyEmailConnection
};

Nodemailer provides transporter.verify() specifically for checking the SMTP connection and authentication before sending mail.

3. Create email service

Create:

backend/services/emailService.js

Add:

const {
  transporter
} = require("../config/email");

const sendOrderConfirmationEmail = async (
  customerEmail,
  customerName,
  order
) => {
  const itemsHtml = order.items
    .map(
      (item) => `
        <li>
          ${item.name}
          × ${item.quantity}
          - ₹${item.price}
        </li>
      `
    )
    .join("");

  await transporter.sendMail({
    from: process.env.EMAIL_FROM,
    to: customerEmail,
    subject: "Order Confirmation",
    html: `
      <h2>Order Confirmed!</h2>

      <p>Hello ${customerName},</p>

      <p>
        Your order has been successfully placed.
      </p>

      <h3>Order Details</h3>

      <p>
        Order ID:
        ${order._id}
      </p>

      <ul>
        ${itemsHtml}
      </ul>

      <p>
        <strong>
          Total: ₹${order.totalAmount}
        </strong>
      </p>

      <p>
        Payment Status:
        ${order.paymentStatus}
      </p>

      <p>
        Order Status:
        ${order.orderStatus}
      </p>

      <p>
        Thank you for shopping with us!
      </p>
    `
  });
};


const sendVendorNewOrderEmail = async (
  vendorEmail,
  vendorName,
  order
) => {
  await transporter.sendMail({
    from: process.env.EMAIL_FROM,
    to: vendorEmail,
    subject: "New Order Received",
    html: `
      <h2>New Order Received</h2>

      <p>Hello ${vendorName},</p>

      <p>
        You have received a new order
        for your store.
      </p>

      <p>
        Order ID:
        ${order._id}
      </p>

      <p>
        Customer:
        ${order.customerId?.name || "Customer"}
      </p>

      <p>
        Total:
        <strong>
          ₹${order.totalAmount}
        </strong>
      </p>

      <p>
        Please log in to your vendor dashboard
        to process the order.
      </p>
    `
  });
};


const sendOrderStatusEmail = async (
  customerEmail,
  customerName,
  order
) => {
  await transporter.sendMail({
    from: process.env.EMAIL_FROM,
    to: customerEmail,
    subject: "Order Status Updated",
    html: `
      <h2>Order Status Updated</h2>

      <p>Hello ${customerName},</p>

      <p>
        Your order status has been updated.
      </p>

      <p>
        Order ID:
        ${order._id}
      </p>

      <p>
        New Status:
        <strong>
          ${order.orderStatus}
        </strong>
      </p>

      <p>
        Thank you for shopping with us!
      </p>
    `
  });
};


module.exports = {
  sendOrderConfirmationEmail,
  sendVendorNewOrderEmail,
  sendOrderStatusEmail
};
4. Send confirmation after order creation

Open:

backend/controllers/orderController.js

At the top, add:

const User = require("../models/User");

const {
  sendOrderConfirmationEmail,
  sendVendorNewOrderEmail
} = require("../services/emailService");

You already use User in some functions with require() inside the function. It's better to have it at the top and remove the repeated:

const User = require("../models/User");

from inside those functions.

5. Update createOrderFromPayment

After:

const order = await Order.create({
  customerId: req.user.userId,
  storeId: storeId,
  items: orderItems,
  totalAmount: totalAmount,
  paymentStatus: "paid",
  orderStatus: "placed",
  stripePaymentId: paymentIntentId
});

add:

const customer =
  await User.findById(
    req.user.userId
  ).select("name email");

const store =
  await Store.findById(
    storeId
  ).populate(
    "owner",
    "name email"
  );

if (customer?.email) {
  try {
    await sendOrderConfirmationEmail(
      customer.email,
      customer.name,
      order
    );
  } catch (emailError) {
    console.error(
      "Customer email failed:",
      emailError.message
    );
  }
}

if (store?.owner?.email) {
  try {
    await sendVendorNewOrderEmail(
      store.owner.email,
      store.owner.name,
      order
    );
  } catch (emailError) {
    console.error(
      "Vendor email failed:",
      emailError.message
    );
  }
}
Why use try/catch around email?

If the payment succeeds but the email server temporarily fails, the customer's order should still remain successful.

So:

Payment successful
       ↓
Order created
       ↓
Email attempted
       ↓
Email fails
       ↓
Order still exists

That's much safer than making an email failure turn a successful purchase into an error.

6. Send email when vendor updates status

Open your existing:

backend/controllers/orderController.js

At the top, also add:

const {
  sendOrderStatusEmail
} = require("../services/emailService");

Then in updateOrderStatus, after:

await order.save();

add:

const customer =
  await User.findById(
    order.customerId
  ).select("name email");

if (customer?.email) {
  try {
    await sendOrderStatusEmail(
      customer.email,
      customer.name,
      order
    );
  } catch (emailError) {
    console.error(
      "Status email failed:",
      emailError.message
    );
  }
}

Then your existing response remains:

res.status(200).json({
  message:
    "Order status updated successfully",
  order
});
7. Test email configuration

Temporarily add this to server.js:

const {
  verifyEmailConnection
} = require("./config/email");

Then after:

dotenv.config();

you can call:

verifyEmailConnection();

Start:

cd backend
npm run dev

You should see:

MongoDB connected: ...
Email server is ready
Server running on port 5000

If you see an authentication error, check:

SMTP_SERVICE
SMTP_USER
SMTP_PASS
8. Test the complete email flow
Customer order
Customer
   ↓
Shop
   ↓
Cart
   ↓
Stripe
   ↓
Payment successful
   ↓
Order created
   ↓
📧 Customer confirmation
   ↓
📧 Vendor new-order notification
Vendor status update
Vendor
   ↓
Orders
   ↓
Select "Processing"
   ↓
Backend updates MongoDB
   ↓
📧 Customer receives:
"Order Status Updated"

Then:

Processing
     ↓
Shipped
     ↓
Delivered

Each change can trigger another email.

9. Your backend structure now

Your project becomes:

backend/
├── config/
│   ├── db.js
│   ├── cloudinary.js
│   ├── stripe.js
│   └── email.js              ← NEW
│
├── controllers/
│   ├── authController.js
│   ├── storeController.js
│   ├── productController.js
│   ├── uploadController.js
│   ├── cartController.js
│   ├── orderController.js
│   ├── paymentController.js
│   ├── analyticsController.js
│   └── adminController.js
│
├── services/
│   └── emailService.js        ← NEW
│
├── routes/
│   ├── authRoutes.js
│   ├── userRoutes.js
│   ├── storeRoutes.js
│   ├── productRoutes.js
│   ├── uploadRoutes.js
│   ├── cartRoutes.js
│   ├── orderRoutes.js
│   ├── paymentRoutes.js
│   ├── analyticsRoutes.js
│   └── adminRoutes.js
│
├── models/
├── middleware/
├── utils/
├── .env
├── .gitignore
└── server.js

















