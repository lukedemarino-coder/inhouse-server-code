// index.js
require("dotenv").config();
const fs = require("fs");
const axios = require("axios");
const cheerio = require("cheerio");
const {
  Client,
  GatewayIntentBits,
  Partials,
  ActionRowBuilder,
  ButtonBuilder,
  ButtonStyle,
  EmbedBuilder,
} = require("discord.js");

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent,
    GatewayIntentBits.GuildVoiceStates, // <-- required for voiceStateUpdate
  ],
  partials: [Partials.Message, Partials.Channel, Partials.Reaction],
});

// ---------------- CONFIG ----------------
const DATA_FILE = "playerData.json";
const QUEUE_SIZE = 10;
let queue = [];
let matches = {};
let playerData = loadData();
let queueMessage;
let leaderboardMessage;

// ---------------- RANK SYSTEM ----------------
const ranks = ["Iron", "Bronze", "Silver", "Gold", "Platinum", "Emerald", "Diamond"];

function getIHP(player) {
  if (["Master", "Grandmaster", "Challenger"].includes(player.rank)) {
    return 2800 + player.lp;
  }
  const tierIndex = ranks.indexOf(player.rank);
  if (tierIndex < 0) return 0;
  let divisionValue = 0;
  if (player.division) {
    divisionValue = (5 - player.division) * 100 - 100;
  }
  return tierIndex * 400 + divisionValue + player.lp;
}

function IHPToRank(IHP) {
  if (IHP >= 2800) {
    // Master+
    const lp = IHP - 2800;
    if (lp >= 900) return { rank: "Challenger", division: null, lp };
    if (lp >= 500) return { rank: "Grandmaster", division: null, lp };
    return { rank: "Master", division: null, lp };
  }
  const tierIndex = Math.floor(IHP / 400);
  const tier = ranks[tierIndex];
  let remainingIHP = IHP - tierIndex * 400;
  let division = 4;
  let lp = remainingIHP;
  while (lp >= 100 && division > 1) {
    lp -= 100;
    division--;
  }
  // Promote tier if division < 1
  if (division < 1) {
    const tierIndex = ranks.indexOf(tier);
    if (tierIndex + 1 < ranks.length) {
      tier = ranks[tierIndex + 1];
      division = 4;
    } else {
      // max tier reached, division stays 1
      division = 1;
    }
  }
  return { rank: tier, division, lp };
}

// ---------------- DATA ----------------
function loadData() {
  if (fs.existsSync(DATA_FILE)) return JSON.parse(fs.readFileSync(DATA_FILE));
  return {};
}

function saveData() {
  fs.writeFileSync(DATA_FILE, JSON.stringify(playerData, null, 2));
}

function ensurePlayer(id) {
  if (!playerData[id]) {
    playerData[id] = { rank: "Iron", division: 4, lp: 0, wins: 0, losses: 0 };
  }
  return playerData[id];
}

client.on("voiceStateUpdate", async (oldState, newState) => {
  const member = newState.member;
  if (!matches.current) return;
  const { team1, team2, team1VC, team2VC } = matches.current;
  const newVC = newState.channelId;
  if (!newVC) return; // left VC
  let allowedTeam = null;
  if (newVC === team1VC.id) allowedTeam = team1;
  if (newVC === team2VC.id) allowedTeam = team2;
  try {
    if (allowedTeam) {
      // user is in a match VC
      if (!allowedTeam.includes(member.id)) {
        if (!newState.serverMute) await newState.setMute(true, "Not on this team");
      } else {
        if (newState.serverMute) await newState.setMute(false, "Player on correct team");
      }
    } else {
      // user is in some other VC
      if (newState.serverMute) await newState.setMute(false, "Not a match VC");
    }
  } catch (err) {
    console.error("Failed to mute/unmute member:", err);
  }
});

// ---------------- QUEUE EMBED ----------------
async function postQueueMessage(channel) {
  const row = new ActionRowBuilder().addComponents(
    new ButtonBuilder().setCustomId("join").setLabel("✅ Join Queue").setStyle(ButtonStyle.Success),
    new ButtonBuilder().setCustomId("leave").setLabel("🚪 Leave Queue").setStyle(ButtonStyle.Danger)
  );
  const embed = buildQueueEmbed();
  queueMessage = await channel.send({ embeds: [embed], components: [row] });
}

function buildQueueEmbed() {
  const playerList = queue.length > 0 ? queue.map((id, i) => `${i + 1}. <@${id}>`).join("\n") : "No players in queue.";
  return new EmbedBuilder()
    .setTitle("League Inhouse Queue")
    .setDescription(`Use the buttons below to join or leave the queue.\n\n + **Current:** ${queue.length}/${QUEUE_SIZE}\n\n${playerList}`);
}

async function updateQueueMessage() {
  if (!queueMessage) return;
  const embed = buildQueueEmbed();
  await queueMessage.edit({ embeds: [embed], components: queueMessage.components });
}

// ---------------- LEADERBOARD ----------------
async function updateLeaderboardChannel(guild) {
  let channel = guild.channels.cache.find((c) => c.name === "leaderboard");
  if (!channel) {
    channel = await guild.channels.create({ name: "leaderboard", type: 0 });
  }
  if (!leaderboardMessage) {
    leaderboardMessage = await channel.send({ content: "Loading leaderboard..." });
  }
  setInterval(() => {
    const players = Object.entries(playerData).map(([id, p]) => {
      const games = (p.wins || 0) + (p.losses || 0);
      const wr = games > 0 ? ((p.wins / games) * 100).toFixed(1) : "0.0";
      return { id, ...p, IHP: getIHP(p), games, wr };
    });
    // Sort by IHP descending
    players.sort((a, b) => b.IHP - a.IHP);
    const embed = new EmbedBuilder()
      .setTitle("🏆 Leaderboard")
      .setColor(0x808080)
      .setTimestamp();
    // Add each player as a separate field
    players.forEach((p, i) => {
      const rankText = p.division ? `${p.rank} ${p.division} ${p.lp} LP` : `${p.rank} ${p.lp} LP`;
      embed.addFields({
        name: `**${i + 1}. <@${p.id}> - ${rankText}** | Elo: ${p.IHP} | W: ${p.wins || 0} | L: ${p.losses || 0} | WR: ${p.wr}% | GP: ${p.games}`,
        value: "\u200B",
        inline: false,
      });
    });
    leaderboardMessage.edit({ embeds: [embed] });
  }, 30000);
}

// ---------------- BUTTON HANDLING ----------------
client.on("interactionCreate", async (interaction) => {
  if (!interaction.isButton()) return;
  const id = interaction.user.id;
  ensurePlayer(id);
  if (interaction.customId === "join") {
    if (queue.includes(id)) return interaction.reply({ content: "Already in queue.", ephemeral: true });
    queue.push(id);
    saveData();
    await interaction.reply({ content: `You joined! (${queue.length}/${QUEUE_SIZE})`, ephemeral: true });
    await updateQueueMessage();
    if (queue.length >= QUEUE_SIZE) makeTeams(interaction.channel);
  }
  if (interaction.customId === "leave") {
    if (!queue.includes(id)) return interaction.reply({ content: "Not in queue.", ephemeral: true });
    queue = queue.filter((x) => x !== id);
    saveData();
    await interaction.reply({ content: `You left. (${queue.length}/${QUEUE_SIZE})`, ephemeral: true });
    await updateQueueMessage();
  }
  if (interaction.customId === "endmatch") {
    return endMatch(interaction.channel, "manual");
  }
});

// ---------------- COMMANDS ----------------
client.on("messageCreate", async (message) => {
  if (message.author.bot) return;
  const [cmd, ...args] = message.content.trim().split(/\s+/);

  if (cmd === "!forcejoin") {
    const id = args[0]?.replace(/[<@!>]/g, "");
    if (!id) return message.reply("Usage: !forcejoin <@user>");
    ensurePlayer(id);
    if (!queue.includes(id)) queue.push(id);
    saveData();
    message.channel.send(`<@${id}> has been added. (${queue.length}/${QUEUE_SIZE})`);
    await updateQueueMessage();
  }

  if (cmd === "!clearqueue") {
    if (!message.member.permissions.has("ManageGuild")) {
      return message.reply("❌ You need Manage Server permissions to do that.");
    }
    queue = [];
    saveData();
    message.channel.send("✅ Queue has been cleared.");
    await updateQueueMessage();
  }

  if (cmd === "!simulate") {
    const count = parseInt(args[0] || "10");
    for (let i = 0; i < count; i++) {
      const fakeId = `fake${i}`;
      ensurePlayer(fakeId);
      if (!queue.includes(fakeId)) queue.push(fakeId);
    }
    saveData();
    message.channel.send(`🤖 Simulated ${count} fake players. Queue = ${queue.length}/${QUEUE_SIZE}`);
    await updateQueueMessage();
  }

  if (cmd === "!endmatch") {
    const team = args[0];
    if (!team || !matches.current) return message.reply("Usage: !endmatch <1|2>");
    endMatch(message.channel, team);
  }

  // ---------------- !forceregister ----------------
  if (message.content.startsWith("!forceregister")) {
    if (!message.member.permissions.has("ManageGuild")) {
      return message.reply("❌ You need Manage Server permissions to do that.");
    }
    const args = message.content.split(" ");
    const userMention = args[1];
    const url = args[2];
    if (!userMention || !url) {
      return message.reply("Usage: !forceregister <@user> <OP.GG link>");
    }
    if (!url.includes("op.gg")) return message.reply("❌ Please provide a valid OP.GG link.");
    const userId = userMention.replace(/[<@!>]/g, "");
    if (!userId) return message.reply("❌ Invalid user mention");
    try {
      const res = await axios.get(url);
      const $ = cheerio.load(res.data);
      const tierText = $("strong.text-xl").first().text().trim();
      const lpText = $("span.text-xs.text-gray-500").first().text().trim();
      const lp = parseInt(lpText);
      if (!tierText || isNaN(lp)) return message.reply("❌ Could not parse rank/LP from OP.GG.");
      let rank, division;
      const tierParts = tierText.trim().split(/\s+/);
      if (tierParts.length === 2) {
        rank = tierParts[0].charAt(0).toUpperCase() + tierParts[0].slice(1).toLowerCase();
        const divText = tierParts[1].toUpperCase();
        const romanToNumber = { "IV": 4, "III": 3, "II": 2, "I": 1 };
        division = !isNaN(parseInt(divText)) ? parseInt(divText) : romanToNumber[divText] || 4;
      } else {
        rank = tierParts[0].charAt(0).toUpperCase() + tierParts[0].slice(1).toLowerCase();
        division = null;
      }
      ensurePlayer(userId);
      playerData[userId].summonerName = url;
      playerData[userId].rank = rank;
      playerData[userId].division = division;
      playerData[userId].lp = lp;
      playerData[userId].IHP = getIHP(playerData[userId]);
      saveData();
      return message.reply(`✅ Force-registered <@${userId}> as **${tierText} ${lp} LP**`);
    } catch (err) {
      console.error(err);
      return message.reply("❌ Failed to fetch OP.GG page. Make sure the link is correct.");
    }
  }
});

// ---------------- !register ----------------
client.on("messageCreate", async (message) => {
  if (message.author.bot) return;
  if (message.content.startsWith("!register")) {
    const userId = message.author.id;
    if (playerData[userId] && playerData[userId].summonerName) {
      return message.reply("❌ You have already registered an account.");
    }
    const parts = message.content.split(" ");
    if (parts.length < 2) return message.reply("Usage: !register <OP.GG link>");
    const url = parts[1];
    if (!url.includes("op.gg")) return message.reply("❌ Please provide a valid OP.GG link.");
    try {
      const res = await axios.get(url);
      const $ = cheerio.load(res.data);
      const tierText = $("strong.text-xl").first().text().trim();
      const lpText = $("span.text-xs.text-gray-500").first().text().trim();
      const lp = parseInt(lpText);
      if (!tierText || isNaN(lp)) return message.reply("❌ Could not parse rank/LP from OP.GG.");
      let rank, division;
      const tierParts = tierText.trim().split(/\s+/);
      if (tierParts.length === 2) {
        rank = tierParts[0].charAt(0).toUpperCase() + tierParts[0].slice(1).toLowerCase();
        const divText = tierParts[1].toUpperCase();
        const romanToNumber = { "IV": 4, "III": 3, "II": 2, "I": 1 };
        if (!isNaN(parseInt(divText))) {
          division = parseInt(divText);
        } else {
          division = romanToNumber[divText] || 4;
        }
      } else {
        rank = tierParts[0].charAt(0).toUpperCase() + tierParts[0].slice(1).toLowerCase();
        division = null;
      }
      ensurePlayer(userId);
      playerData[userId].summonerName = url;
      playerData[userId].rank = rank;
      playerData[userId].division = division;
      playerData[userId].lp = lp;
      playerData[userId].IHP = getIHP(playerData[userId]);
      saveData();
      return message.reply(`✅ Registered ${message.author.username} as **${tierText} ${lp} LP**`);
    } catch (err) {
      console.error(err);
      return message.reply("❌ Failed to fetch OP.GG page. Make sure the link is correct.");
    }
  }
});

// ---------------- MATCHMAKING ----------------
async function makeTeams(channel) {
  const players = [...queue];
  const IHPs = players.map((id) => getIHP(ensurePlayer(id)));
  let bestTeam1 = null;
  let bestTeam2 = null;
  let bestDiff = Infinity;

  function combine(arr, k, start = 0, path = []) {
    if (path.length === k) {
      const team1 = path;
      const team2 = arr.filter((_, i) => !path.includes(i));
      const sum1 = team1.reduce((s, idx) => s + IHPs[idx], 0);
      const sum2 = team2.reduce((s, idx) => s + IHPs[idx], 0);
      const diff = Math.abs(sum1 - sum2);
      if (diff < bestDiff) {
        bestDiff = diff;
        bestTeam1 = team1.map((i) => players[i]);
        bestTeam2 = team2.map((i) => players[i]);
      }
      return;
    }
    for (let i = start; i < arr.length; i++) {
      combine(arr, k, i + 1, [...path, i]);
    }
  }
  combine(players.map((_, i) => i), 5);

  const avg1 = Math.round(bestTeam1.reduce((a, id) => a + getIHP(ensurePlayer(id)), 0) / 5);
  const avg2 = Math.round(bestTeam2.reduce((a, id) => a + getIHP(ensurePlayer(id)), 0) / 5);

  // ---------------- CREATE MATCH CATEGORY + CHANNELS ----------------
  const guild = channel.guild;
  let matchCategory = guild.channels.cache.find(
    c => c.type === 4 && c.name.toLowerCase() === "match"
  );
  if (!matchCategory) {
    matchCategory = await guild.channels.create({ name: "Match", type: 4 });
  }

  const matchChannel = await guild.channels.create({
    name: "match",
    type: 0,
    parent: matchCategory.id
  });

  const team1VC = await guild.channels.create({
    name: "Team 1 (Blue Side)",
    type: 2,
    parent: matchCategory.id
  });

  const team2VC = await guild.channels.create({
    name: "Team 2 (Red Side)",
    type: 2,
    parent: matchCategory.id
  });

  const embed = new EmbedBuilder()
    .setTitle("🎮 Match Ready!")
    .addFields(
      { name: `Team 1 (Avg Elo: ${avg1})`, value: bestTeam1.map((id) => formatPlayer(id)).join("\n"), inline: true },
      { name: `Team 2 (Avg Elo: ${avg2})`, value: bestTeam2.map((id) => formatPlayer(id)).join("\n"), inline: true }
    )
    .setFooter({ text: "Use the team voice channels below to join your squad!" });

  await matchChannel.send({ embeds: [embed] });
  matches.current = { team1: bestTeam1, team2: bestTeam2, matchChannel, team1VC, team2VC };
  queue = [];
  saveData();
  updateQueueMessage();
}

function formatPlayer(id) {
  const p = ensurePlayer(id);
  if (p.division) return `<@${id}> - ${p.rank} ${p.division} ${p.lp} LP (${getIHP(p)})`;
  return `<@${id}> - ${p.rank} ${p.lp} LP (${getIHP(p)})`;
}

// ---------------- END MATCH ----------------
async function endMatch(channel, winner) {
  if (!matches.current) return channel.send("No active match.");
  const { team1, team2, matchChannel, team1VC, team2VC } = matches.current;
  const guild = channel.guild;

  const general = guild.channels.cache.find(c => c.name === "general" && c.type === 0);

  let historyChannel = guild.channels.cache.find(c => c.name === "history" && c.type === 0);
  if (!historyChannel) {
    historyChannel = await guild.channels.create({ name: "history", type: 0 });
  }

  const winners = winner === "1" ? team1 : team2;
  const losers = winner === "1" ? team2 : team1;

  if (winners.includes("272603932268822529")) {
    if (general) general.send("Wow Romeo actually won? That guy is ass");
  }

  winners.forEach((id) => {
    const p = ensurePlayer(id);
    const oldIHP = getIHP(p);
    const oldRank = p.rank;
    const oldDivision = p.division;
    p.wins++;
    p.lp += 20;
    checkRankChange(id, general, p, oldIHP, oldRank, oldDivision);
  });

  losers.forEach((id) => {
  const p = ensurePlayer(id);
  const oldIHP = getIHP(p);
  const oldRank = p.rank;
  const oldDivision = p.division;

  p.losses++;
  p.lp -= 20; // subtract LP

  // Clamp LP temporarily if you want to allow negative for demotion
  if (p.lp < 0) {
    // Demotion logic will handle negative LP correctly in IHPToRank
  }

  checkRankChange(id, general, p, oldIHP, oldRank, oldDivision);

  // Ensure LP is at least 0 for display purposes
  if (p.lp < 0) p.lp = 0;
});

  saveData();

  const team1Players = team1.map(id => `<@${id}>`).join(", ") || "—";
  const team2Players = team2.map(id => `<@${id}>`).join(", ") || "—";

  const embed = new EmbedBuilder()
    .setTitle("📜 Match History")
    .setDescription(`**Winner:** ${winner === "1" ? "🟦 Team 1 (Blue)" : "🟥 Team 2 (Red)"}`)
    .addFields(
      { name: "🟦 Team 1 (Blue)", value: team1Players, inline: false },
      { name: "🟥 Team 2 (Red)", value: team2Players, inline: false }
    )
    .setTimestamp();

  await historyChannel.send({ embeds: [embed] });

  try {
    if (matchChannel) await matchChannel.delete();
    if (team1VC) await team1VC.delete();
    if (team2VC) await team2VC.delete();
    const matchCategory = guild.channels.cache.find(c => c.type === 4 && c.name.toLowerCase() === "match");
    if (matchCategory) await matchCategory.delete();
  } catch (err) {
    console.error("Error deleting match channels:", err);
  }

  matches.current = null;
}

function checkRankChange(id, announceChannel, player, oldIHP, oldRank, oldDivision) {
  const newStats = IHPToRank(getIHP(player), oldRank, oldDivision);
  Object.assign(player, newStats);
  const newIHP = getIHP(player);
  if (player.rank !== oldRank || player.division !== oldDivision) {
    if (newIHP > oldIHP) {
      announceChannel.send(`🎉 <@${id}> ranked up to **${player.rank}${player.division ? " " + player.division : ""}**! 🏅`);
    } else if (newIHP < oldIHP) {
      announceChannel.send(`⬇️ <@${id}> has been demoted to **${player.rank}${player.division ? " " + player.division : ""}**.`);
    }
  }
}

// ---------------- READY ----------------
const MAIN_GUILD_ID = "1421221145532956722";
client.once("ready", async () => {
  console.log(`✅ Logged in as ${client.user.tag}`);
  const guild = client.guilds.cache.get(MAIN_GUILD_ID);
  if (!guild) {
    console.log("Bot is not in the main server!");
    return;
  }
  let queueChannel = guild.channels.cache.find((c) => c.name === "queue");
  if (!queueChannel) {
    queueChannel = await guild.channels.create({ name: "queue", type: 0 });
  }
  await postQueueMessage(queueChannel);
  await updateLeaderboardChannel(guild);
});

client.login(process.env.BOT_TOKEN);
