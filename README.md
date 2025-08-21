Backend (Node.js + MongoDB)
backend/package.json
{
  "name": "securin-recipes-backend",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "load": "node scripts/load_json.js"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "mongodb": "^6.8.0",
    "mongoose": "^8.5.1",
    "morgan": "^1.10.0"
  },
  "devDependencies": {
    "nodemon": "^3.1.0"
  }
}

backend/.env.example
MONGO_URI=mongodb://localhost:27017/recipesdb
PORT=5000

backend/server.js
import express from "express";
import mongoose from "mongoose";
import cors from "cors";
import morgan from "morgan";
import dotenv from "dotenv";
import recipesRouter from "./routes/recipes.js";

dotenv.config();
const app = express();
app.use(cors());
app.use(express.json());
app.use(morgan("dev"));

app.get("/health", (_req, res) => res.json({ ok: true }));
app.use("/api/recipes", recipesRouter);

const MONGO_URI = process.env.MONGO_URI || "mongodb://localhost:27017/recipesdb";
const PORT = process.env.PORT || 5000;

mongoose.connect(MONGO_URI).then(() => {
  console.log("✅ MongoDB connected");
  app.listen(PORT, () => console.log(`🚀 Server running on http://localhost:${PORT}`));
});

backend/models/Recipe.js
import mongoose from "mongoose";

const NutrientsSchema = new mongoose.Schema({}, { strict: false });

const RecipeSchema = new mongoose.Schema({
  cuisine: String,
  title: { type: String, required: true },
  rating: Number,
  prep_time: Number,
  cook_time: Number,
  total_time: Number,
  description: String,
  nutrients: NutrientsSchema,
  serves: String
});

export default mongoose.model("Recipe", RecipeSchema);

backend/routes/recipes.js
import { Router } from "express";
import Recipe from "../models/Recipe.js";

const router = Router();

// GET /api/recipes?page&limit
router.get("/", async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 10;
  const skip = (page - 1) * limit;

  const total = await Recipe.countDocuments();
  const data = await Recipe.find().sort({ rating: -1 }).skip(skip).limit(limit);

  res.json({ page, limit, total, data });
});

// GET /api/recipes/search
router.get("/search", async (req, res) => {
  const { title, cuisine, rating, total_time } = req.query;
  let query = {};

  if (title) query.title = { $regex: title, $options: "i" };
  if (cuisine) query.cuisine = { $regex: cuisine, $options: "i" };
  if (rating) query.rating = { $gte: parseFloat(rating) };
  if (total_time) query.total_time = { $lte: parseInt(total_time) };

  const data = await Recipe.find(query).sort({ rating: -1 });
  res.json({ data });
});

export default router;

backend/scripts/load_json.js
import fs from "fs";
import mongoose from "mongoose";
import dotenv from "dotenv";
import Recipe from "../models/Recipe.js";

dotenv.config();
const MONGO_URI = process.env.MONGO_URI || "mongodb://localhost:27017/recipesdb";
const filePath = "./data/US_recipes.json";

function toNull(v) {
  return isNaN(v) ? null : v;
}

async function main() {
  await mongoose.connect(MONGO_URI);
  const raw = fs.readFileSync(filePath, "utf-8");
  const recipes = JSON.parse(raw);

  const docs = recipes.map(r => ({
    cuisine: r.cuisine,
    title: r.title,
    rating: toNull(r.rating),
    prep_time: toNull(r.prep_time),
    cook_time: toNull(r.cook_time),
    total_time: toNull(r.total_time),
    description: r.description,
    nutrients: r.nutrients,
    serves: r.serves
  }));

  await Recipe.deleteMany({});
  await Recipe.insertMany(docs);
  console.log("✅ Data loaded into MongoDB");
  mongoose.disconnect();
}

main();

🔹 Frontend (React + Vite + MUI)
frontend/package.json
{
  "name": "securin-recipes-frontend",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "@mui/material": "^5.16.7",
    "@mui/icons-material": "^5.16.7",
    "axios": "^1.7.4",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.1",
    "vite": "^5.4.2"
  }
}

frontend/vite.config.js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      "/api": "http://localhost:5000"
    }
  }
})

frontend/index.html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Recipes</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>

frontend/src/main.jsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App.jsx";

ReactDOM.createRoot(document.getElementById("root")).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);

frontend/src/App.jsx
import React, { useEffect, useState } from "react";
import axios from "axios";
import {
  Table, TableBody, TableCell, TableContainer, TableHead, TableRow,
  Paper, Drawer, Typography, Rating
} from "@mui/material";

export default function App() {
  const [recipes, setRecipes] = useState([]);
  const [page, setPage] = useState(1);
  const [selected, setSelected] = useState(null);

  useEffect(() => {
    axios.get(`/api/recipes?page=${page}&limit=10`)
      .then(res => setRecipes(res.data.data));
  }, [page]);

  return (
    <div style={{ padding: 20 }}>
      <Typography variant="h4">Recipes</Typography>
      <TableContainer component={Paper}>
        <Table>
          <TableHead>
            <TableRow>
              <TableCell>Title</TableCell>
              <TableCell>Cuisine</TableCell>
              <TableCell>Rating</TableCell>
              <TableCell>Total Time</TableCell>
              <TableCell>Serves</TableCell>
            </TableRow>
          </TableHead>
          <TableBody>
            {recipes.map(r => (
              <TableRow key={r._id} onClick={() => setSelected(r)} style={{ cursor: "pointer" }}>
                <TableCell>{r.title}</TableCell>
                <TableCell>{r.cuisine}</TableCell>
                <TableCell><Rating value={r.rating} readOnly precision={0.1} /></TableCell>
                <TableCell>{r.total_time}</TableCell>
                <TableCell>{r.serves}</TableCell>
              </TableRow>
            ))}
          </TableBody>
        </Table>
      </TableContainer>

      <Drawer anchor="right" open={!!selected} onClose={() => setSelected(null)}>
        {selected && (
          <div style={{ width: 400, padding: 20 }}>
            <Typography variant="h6">{selected.title} - {selected.cuisine}</Typography>
            <Typography><b>Description:</b> {selected.description}</Typography>
            <Typography><b>Total Time:</b> {selected.total_time}</Typography>
            <Typography><b>Prep Time:</b> {selected.prep_time}</Typography>
            <Typography><b>Cook Time:</b> {selected.cook_time}</Typography>
          </div>
        )}
      </Drawer>
    </div>
  );
}
