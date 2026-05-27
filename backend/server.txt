const express = require("express");
const mongoose = require("mongoose");
const cors = require("cors");
const path = require("path");
require("dotenv").config();

const app = express();

app.use(cors());
app.use(express.json());

mongoose
  .connect(process.env.MONGO_URI)
  .then(() => {
    console.log("Connected to MongoDB");
    const PORT = process.env.PORT || 3001;

    app.listen(PORT, () => {
      console.log(`Server running on http://localhost:${PORT}`);
    });
  })
  .catch((err) => {
    console.error("Connection Error:", err);
    process.exit(1);
  });

const userSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: [true, "Tên không được để trống"],
      minlength: [2, "Tên phải có ít nhất 2 ký tự"],
      trim: true,
    },
    age: {
      type: Number,
      required: [true, "Tuổi không được để trống"],
      min: [0, "Tuổi phải lớn hơn hoặc bằng 0"],
    },
    email: {
      type: String,
      trim: true,
      lowercase: true,
      unique: true,
      match: [/^\S+@\S+\.\S+$/, "Email không hợp lệ"],
      required: [true, "Email không được để trống"],
    },
    address: {
      type: String,
      required: false,
      trim: true,
    },
  },
  {
    collection: "users-collection",
  }
);

const User = mongoose.model("User", userSchema);

// Tạo/đồng bộ index để unique email có hiệu lực
User.syncIndexes().catch((err) => {
  console.error("Sync index error:", err.message);
});

app.use((req, res, next) => {
  console.log(req.method, req.url);
  next();
});

// GET /api/users?page=1&limit=5&search=abc
app.get("/api/users", async (req, res) => {
  try {
    let { page = 1, limit = 5, search = "" } = req.query;
    page = Math.max(parseInt(page, 10) || 1, 1);
    limit = Math.min(Math.max(parseInt(limit, 10) || 5, 1), 50);
    search = String(search).trim();

    const query = search
      ? {
          $or: [
            { name: { $regex: search, $options: "i" } },
            { email: { $regex: search, $options: "i" } },
            { address: { $regex: search, $options: "i" } },
          ],
        }
      : {};

    const [total, data] = await Promise.all([
      User.countDocuments(query),
      User.find(query)
        .skip((page - 1) * limit)
        .limit(limit),
    ]);

    const totalPages = Math.max(1, Math.ceil(total / limit));
    res.json({ page, limit, total, totalPages, data });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "Server error" });
  }
});

// POST /api/users
app.post("/api/users", async (req, res) => {
  try {
    const { name = "", age, email = "", address = "" } = req.body;

    const cleanEmail = String(email).trim().toLowerCase();

    const existedUser = await User.findOne({ email: cleanEmail });
    if (existedUser) {
      return res.status(400).json({ error: "Email đã tồn tại" });
    }

    const newUser = new User({
      name: String(name).trim(),
      age: Number(age),
      email: cleanEmail,
      address: address ? String(address).trim() : undefined,
    });

    await newUser.save();
    res.status(201).json(newUser);
  } catch (err) {
    console.error(err);

    if (err.code === 11000) {
      return res.status(400).json({ error: "Email đã tồn tại" });
    }

    if (err.name === "ValidationError") {
      return res.status(400).json({ error: err.message });
    }

    res.status(500).json({ error: "Server error" });
  }
});

// PUT /api/users/:id
app.put("/api/users/:id", async (req, res) => {
  try {
    const { id } = req.params;
    if (!mongoose.Types.ObjectId.isValid(id)) {
      return res.status(400).json({ error: "ID không hợp lệ" });
    }

    const updateData = {};
    const allowedFields = ["name", "age", "email", "address"];

    for (const field of allowedFields) {
      if (req.body[field] !== undefined) {
        updateData[field] =
          typeof req.body[field] === "string"
            ? req.body[field].trim()
            : req.body[field];
      }
    }

    if (updateData.age !== undefined) {
      updateData.age = Number(updateData.age);
    }

    if (updateData.email !== undefined) {
      updateData.email = String(updateData.email).trim().toLowerCase();

      const existedUser = await User.findOne({
        email: updateData.email,
        _id: { $ne: id },
      });

      if (existedUser) {
        return res.status(400).json({ error: "Email đã tồn tại" });
      }
    }

    const updatedUser = await User.findByIdAndUpdate(id, updateData, {
      new: true,
      runValidators: true,
    });

    if (!updatedUser) {
      return res.status(404).json({ error: "Người dùng không tồn tại" });
    }

    res.json(updatedUser);
  } catch (err) {
    console.error(err);

    if (err.code === 11000) {
      return res.status(400).json({ error: "Email đã tồn tại" });
    }

    if (err.name === "ValidationError") {
      return res.status(400).json({ error: err.message });
    }

    res.status(500).json({ error: "Server error" });
  }
});

// DELETE /api/users/:id
app.delete("/api/users/:id", async (req, res) => {
  try {
    const { id } = req.params;
    if (!mongoose.Types.ObjectId.isValid(id)) {
      return res.status(400).json({ error: "ID không hợp lệ" });
    }

    const deletedUser = await User.findByIdAndDelete(id);
    if (!deletedUser) {
      return res.status(404).json({ error: "Người dùng không tồn tại" });
    }

    res.json({ message: "Xóa người dùng thành công" });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "Server error" });
  }
});

app.use(express.static(path.join(__dirname, "../frontend")));
