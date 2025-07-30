# debate-tab
debate-tab-system/
├── models/
│   ├── Team.js
│   └── Round.js
├── routes/
│   ├── teams.js
│   ├── rounds.js
│   └── tab.js
└── server.js

import mongoose from "mongoose";

const teamSchema = new mongoose.Schema({
  name: String,
  institution: String,
  speakers: [String]
});

export default mongoose.model("Team", teamSchema);

import mongoose from "mongoose";

const matchupSchema = new mongoose.Schema({
  teamA: { type: mongoose.Schema.Types.ObjectId, ref: "Team" },
  teamB: { type: mongoose.Schema.Types.ObjectId, ref: "Team" },
  winner: { type: mongoose.Schema.Types.ObjectId, ref: "Team" },
  scores: {
    type: Map,
    of: [Number]
  }
});

const roundSchema = new mongoose.Schema({
  round: Number,
  matchups: [matchupSchema]
});

export default mongoose.model("Round", roundSchema);

import express from "express";
import Team from "../models/Team.js";

const router = express.Router();

router.get("/", async (req, res) => {
  const teams = await Team.find();
  res.json(teams);
});

router.post("/", async (req, res) => {
  const team = new Team(req.body);
  await team.save();
  res.status(201).json(team);
});

export default router;

import express from "express";
import Round from "../models/Round.js";

const router = express.Router();

router.get("/", async (req, res) => {
  const rounds = await Round.find().populate("matchups.teamA matchups.teamB matchups.winner");
  res.json(rounds);
});

router.post("/", async (req, res) => {
  const round = new Round(req.body);
  await round.save();
  res.status(201).json(round);
});

export default router;

import express from "express";
import Team from "../models/Team.js";
import Round from "../models/Round.js";

const router = express.Router();

router.get("/team", async (req, res) => {
  const teams = await Team.find();
  const rounds = await Round.find();

  const teamStats = {};
  teams.forEach((team) => {
    teamStats[team._id] = {
      team,
      winCount: 0,
      totalSpeakerScore: 0
    };
  });

  for (const round of rounds) {
    for (const match of round.matchups) {
      const { winner, scores } = match;
      if (winner) teamStats[winner].winCount += 1;
      for (const [teamId, speakerScores] of scores.entries()) {
        const sum = speakerScores.reduce((a, b) => a + b, 0);
        teamStats[teamId].totalSpeakerScore += sum;
      }
    }
  }

  const result = Object.values(teamStats).sort((a, b) => {
    if (b.winCount !== a.winCount) return b.winCount - a.winCount;
    return b.totalSpeakerScore - a.totalSpeakerScore;
  });

  res.json(result);
});

router.get("/speaker", async (req, res) => {
  const rounds = await Round.find();
  const speakerStats = {};

  for (const round of rounds) {
    for (const match of round.matchups) {
      const { scores } = match;
      for (const [teamId, speakerScores] of scores.entries()) {
        const team = await Team.findById(teamId);
        if (!team) continue;

        team.speakers.forEach((name, idx) => {
          if (!speakerStats[name]) speakerStats[name] = { total: 0, count: 0 };
          speakerStats[name].total += speakerScores[idx];
          speakerStats[name].count += 1;
        });
      }
    }
  }

  const result = Object.entries(speakerStats).map(([name, stat]) => ({
    name,
    total: stat.total,
    avg: (stat.total / stat.count).toFixed(2)
  })).sort((a, b) => b.total - a.total);

  res.json(result);
});

export default router;

import express from "express";
import mongoose from "mongoose";
import cors from "cors";
import teamRoutes from "./routes/teams.js";
import roundRoutes from "./routes/rounds.js";
import tabRoutes from "./routes/tab.js";

const app = express();
app.use(cors());
app.use(express.json());

mongoose.connect("mongodb://localhost:27017/tab_system", {
  useNewUrlParser: true,
  useUnifiedTopology: true
});

app.use("/api/teams", teamRoutes);
app.use("/api/rounds", roundRoutes);
app.use("/api/tab", tabRoutes);

const PORT = 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
