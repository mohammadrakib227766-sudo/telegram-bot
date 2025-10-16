# telegram-bot
A Telegram bot built using Python.
// index.js
import TelegramBot from "node-telegram-bot-api";
import mongoose from "mongoose";

// env vars from Replit Secrets
const token = process.env.TOKEN;
const mongoURL = process.env.MONGO_URL;
const ADMIN_ID = process.env.ADMIN_ID || "YOUR_ADMIN_TELEGRAM_ID"; // set in Secrets

// connect mongodb
mongoose.connect(mongoURL, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(()=> console.log("MongoDB connected"))
  .catch(err => console.log("MongoDB error", err));

// Schemas
const userSchema = new mongoose.Schema({
  userId: Number,
  name: String,
  balance: { type: Number, default: 0 },
  lastSeenVideos: { type: Date, default: null },
  referrals: { type: Number, default: 0 }
});
const withdrawSchema = new mongoose.Schema({
  userId: Number,
  method: String,
  account: String,
  amount: Number,
  status: { type: String, default: "pending" }, // pending/paid/rejected
  createdAt: { type: Date, default: Date.now }
});

const User = mongoose.model("User", userSchema);
const Withdraw = mongoose.model("Withdraw", withdrawSchema);

// create bot
const bot = new TelegramBot(token, { polling: true });

bot.onText(/\/start/, async (msg) => {
  const chatId = msg.chat.id;
  let user = await User.findOne({ userId: chatId });
  if (!user) {
    user = new User({ userId: chatId, name: msg.from.first_name || "User" });
    await user.save();
  }

  const keyboard = {
    reply_markup: {
      keyboard: [
        [{ text: "🎬 Watch Videos" }, { text: "📢 Join Tasks" }],
        [{ text: "💰 My Balance" }, { text: "💸 Withdraw" }],
        [{ text: "👥 Refer & Earn" }, { text: "ℹ️ About" }]
      ],
      resize_keyboard: true
    }
  };
  bot.sendMessage(chatId, `👋 স্বাগতম ${user.name}!\nতোমার ব্যালেন্স: ${user.balance} টাকা`, keyboard);
});

// Handle basic menu messages
bot.on("message", async (msg) => {
  const chatId = msg.chat.id;
  const text = msg.text;

  // ignore commands handled elsewhere
  if (!text) return;
  if (text.startsWith("/")) return;

  // Ensure user exists
  let user = await User.findOne({ userId: chatId });
  if (!user) {
    user = new User({ userId: chatId, name: msg.from.first_name || "User" });
    await user.save();
  }

  if (text === "🎬 Watch Videos") {
    // send 10 video links (example). Ideally dynamic from DB.
    const videos = [
      "https://youtu.be/example1",
      "https://youtu.be/example2",
      // ... add up to 10 links
    ];
    let msgText = "🎥 আজকের ১০টি ভিডিও (প্রতি ভিডিও 5 টাকা):\n\n";
    videos.forEach((v,i)=> msgText += `${i+1}. ${v}\n`);
    bot.sendMessage(chatId, msgText);
    // NOTE: you must verify watch externally - here we just suggest and admin can credit manually or use buttons+callbacks
  }

  else if (text === "📢 Join Tasks") {
    const tasks = `Join these to earn 15 টাকা each:\n\n1) YouTube channel: https://youtube.com/@yourchannel\n2) Facebook page: https://facebook.com/yourpage\n3) Telegram group: https://t.me/yourgroup\n\nAfter joining, send proof (screenshot) or press "Check & Claim" if you integrated automatic check.`;
    bot.sendMessage(chatId, tasks);
  }

  else if (text === "💰 My Balance") {
    bot.sendMessage(chatId, `💵 তোমার ব্যালেন্স: ${user.balance} টাকা`);
  }

  else if (text === "💸 Withdraw") {
    const opts = {
      reply_markup: {
        inline_keyboard: [
          [{ text: "📱 Bkash", callback_data: "method_bkash" }],
          [{ text: "💰 Nagad", callback_data: "method_nagad" }],
          [{ text: "🏦 Rocket", callback_data: "method_rocket" }]
        ]
      }
    };
    bot.sendMessage(chatId, "🔰 Withdraw পদ্ধতি নির্বাচন করুন:", opts);
  }

  else if (text === "👥 Refer & Earn") {
    const link = `https://t.me/${bot.username}?start=${chatId}`;
    bot.sendMessage(chatId, `তোমার রেফার লিংক:\n${link}\nপ্রতি রেফারে বোনাস: 40 টাকা (উদাহরণ)`);
  }

  else if (text === "ℹ️ About") {
    bot.sendMessage(chatId, "🔰 RedChilliLite Bot - Daily tasks দিয়ে আয় করো। Admin কে payment request যাবে এবং ম্যানুয়ালি পে হবে।");
  }
});

// Callback queries for withdraw method selection
bot.on("callback_query", async (q) => {
  const chatId = q.message.chat.id;
  const data = q.data;

  if (data.startsWith("method_")) {
    const method = data.split("_")[1]; // bkash, nagad etc
    bot.sendMessage(chatId, `✍️ আপনার ${method.toUpperCase()} নম্বর দিন:`);

    // wait for next message (account number)
    bot.once("message", async (msg2) => {
      // basic validation
      const account = msg2.text;
      bot.sendMessage(chatId, "💵 এখন কত টাকা তুলতে চান লিখুন (সংখ্যায়):");

      bot.once("message", async (msg3) => {
        const amount = parseFloat(msg3.text);
        if (isNaN(amount) || amount <= 0) {
          return bot.sendMessage(chatId, "🚫 সঠিক সংখ্যায় AMOUNT দিন। Withdraw বাদ দিলাম।");
        }

        // save withdraw request
        const wr = new Withdraw({
          userId: chatId,
          method,
          account,
          amount
        });
        await wr.save();

        bot.sendMessage(chatId, `✅ Withdrawal request submit successfully ✅\n\nMethod: ${method.toUpperCase()}\nAccount: ${account}\nAmount: ${amount} taka\nAdmin approval অপেক্ষা করুন।`);

        // notify admin
        const adminMsg = `📢 New Withdraw Request\nUser: ${chatId}\nName: ${msg2.from?.first_name || "User"}\nMethod: ${method}\nAccount: ${account}\nAmount: ${amount}\nRequestID: ${wr._id}`;
        const adminId = ADMIN_ID;
        try {
          await bot.sendMessage(adminId, adminMsg);
        } catch (e) {
          console.log("Admin notify fail:", e.message);
        }
      });
    });
  }
});![IMG-20251005-WA0004](https://github.com/user-attachments/assets/3bfff494-6acd-4a0f-8d8f-1a86de9d6573)
