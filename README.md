import express from "express";
import mongoose from "mongoose";
import cors from "cors";
import dotenv from "dotenv";
import jwt from "jsonwebtoken";
import Post from "./models/Post.js";
import { sendReward } from "./rewardController.js";
import globalPay from "./globalPay.js";

dotenv.config();
const app = express();
app.use(cors());
app.use(express.json());

// MongoDB接続
mongoose.connect(process.env.MONGO_URI || "mongodb://localhost:27017/snsapp");

// JWT認証ミドルウェア
const authenticateToken = (req, res, next) => {
  const token = req.headers["authorization"]?.split(" ")[1];
  if (!token) return res.status(403).json({ message: "Token required" });

  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) return res.status(403).json({ message: "Invalid token" });
    req.user = user;
    next();
  });
};

// 投稿取得
app.get("/posts", async (_, res) => {
  const posts = await Post.find().sort({ createdAt: -1 });
  res.json(posts);
});

// 投稿作成（報酬発行も実行）
app.post("/upload", authenticateToken, async (req, res) => {
  const { username, caption, mediaUrl, wallet } = req.body;
  const post = new Post({ username, caption, mediaUrl });
  await post.save();

  if (wallet) await sendReward(wallet, "post");
  res.json(post);
});

// いいね機能（報酬付与）
app.post("/like/:id", authenticateToken, async (req, res) => {
  const post = await Post.findById(req.params.id);
  post.likes++;
  await post.save();

  const { wallet } = req.body;
  if (wallet) await sendReward(wallet, "like");
  res.json(post);
});

// 送金API
app.use("/pay", globalPay);

app.listen(5000, () => console.log("✅ Server running on http://localhost:5000"));
import mongoose from "mongoose";

const PostSchema = new mongoose.Schema({
  username: String,
  caption: String,
  mediaUrl: String,
  likes: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now },
});

export default mongoose.model("Post", PostSchema);
import { ethers } from "ethers";

const provider = new ethers.providers.JsonRpcProvider(process.env.POLYGON_RPC);
const privateKey = process.env.POLYGON_PRIVATE_KEY!;
export const wallet = new ethers.Wallet(privateKey, provider);

export async function sendReward(address: string, type: string) {
  const rewardAmount = type === "post" ? 0.005 : 0.001;  // 投稿には高い報酬
  const tx = await wallet.sendTransaction({
    to: address,
    value: ethers.utils.parseEther(rewardAmount.toString())
  });
  console.log("Reward sent:", tx.hash);
  return tx.hash;
}
import Commerce from '@chec/commerce.js';

const commerce = new Commerce(process.env.COINBASE_API_KEY!, true);

export async function createPayment(amount: number, currency = "USD") {
  const charge = await commerce.checkout.capture('checkout_token_id', {
    amount,
    currency,
  });
  return charge;
}
import OpenAI from "openai";

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export async function generateCaption(imageUrl: string) {
  const res = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [
      { role: "system", content: "あなたは画像からキャプションを生成するAIです。" },
      { role: "user", content: `この画像に合うSNSキャプションを日本語で書いてください: ${imageUrl}` },
    ],
  });
  return res.choices[0].message?.content || "";
}
SUPABASE_DB_URL=your_supabase_db_url
POLYGON_RPC=your_polygon_rpc
OPENAI_API_KEY=your_openai_api_key
COINBASE_API_KEY=your_coinbase_api_key
JWT_SECRET=your_jwt_secret_key  # セキュアなJWTの秘密鍵
import { useEffect, useState } from "react";
import axios from "axios";
import { PaymentButton } from "../components/PaymentButton";
import { AIComposer } from "../components/AIComposer";

export default function Home() {
  const [posts, setPosts] = useState([]);
  const [caption, setCaption] = useState("");
  const [mediaUrl, setMediaUrl] = useState("");
  const [wallet, setWallet] = useState("");

  useEffect(() => {
    axios.get("/api/posts").then((res) => setPosts(res.data));
  }, []);

  async function submitPost() {
    const token = localStorage.getItem("jwt");  // ローカルストレージにJWT保存
    await axios.post("/api/posts", {
      user: "guest",
      caption,
      mediaUrl,
      wallet,
      headers: { Authorization: `Bearer ${token}` },
    });
    const res = await axios.get("/api/posts");
    setPosts(res.data);
  }

  return (
    <main className="p-6 bg-gray-900 text-white min-h-screen">
      <h1 className="text-3xl font-bold mb-6">NextSNS SuperApp 🚀</h1>
      <AIComposer onCaption={setCaption} />
      <input value={mediaUrl} onChange={(e) => setMediaUrl(e.target.value)} placeholder="画像URL" className="text-black p-2 rounded"/>
      <button onClick={submitPost} className="bg-blue-500 px-4 py-2 ml-2 rounded">投稿</button>

      <div className="grid grid-cols-1 md:grid-cols-2 gap-4 mt-6">
        {posts.map((p) => (
          <div key={p.id} className="bg-gray-800 p-4 rounded">
            <img src={p.mediaUrl} className="rounded w-full"/>
            <p className="mt-2">{p.caption}</p>
          </div>
        ))}
      </div>

      <div className="mt-10">
        <PaymentButton />
      </div>
    </main>
  );
}
import jwt from "jsonwebtoken";

// JWT生成
export function generateToken(userId: string) {
  return jwt.sign({ userId }, process.env.JWT_SECRET, { expiresIn: '1h' });
}

// JWT検証
export function verifyToken(token: string) {
  return jwt.verify(token, process.env.JWT_SECRET);
}
