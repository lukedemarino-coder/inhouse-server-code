// index.js
require("dotenv").config();
const fs = require("fs");
const axios = require("axios");
const cheerio = require("cheerio");
const puppeteer = require("puppeteer");
const {
  Client,
  GatewayIntentBits,
  Partials,
  ActionRowBuilder,
  ButtonBuilder,
  ButtonStyle,
  EmbedBuilder,
  StringSelectMenuBuilder, 
  StringSelectMenuOptionBuilder
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
let activeReadyCheck = null;

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
  // Prevent going below the lowest possible value (Iron IV 0 LP)
  if (IHP <= 0) {
    return { rank: "Iron", division: 4, lp: 0 };
  }

  if (IHP >= 2800) {
    // Master+
    const lp = IHP - 2800;
    if (lp >= 900) return { rank: "Challenger", division: null, lp };
    if (lp >= 500) return { rank: "Grandmaster", division: null, lp };
    return { rank: "Master", division: null, lp };
  }

  const tierIndex = Math.floor(IHP / 400);
  const tier = ranks[tierIndex] || "Iron"; // default to Iron if somehow undefined
  let remainingIHP = IHP - tierIndex * 400;
  let division = 4;
  let lp = remainingIHP;

  while (lp >= 100 && division > 1) {
    lp -= 100;
    division--;
  }

  // Safety: if anything goes negative, reset to base Iron IV 0 LP
  if (lp < 0) lp = 0;

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

// ---------------- Role Selection -------------
const ROLES = [
  { label: "Top", value: "top" },
  { label: "Jungle", value: "jungle" },
  { label: "Mid", value: "mid" },
  { label: "ADC", value: "adc" },
  { label: "Support", value: "support" },
];

async function sendRoleSelection(user) {
  const rows = [];

  for (let i = 0; i < 5; i++) {
    const select = new StringSelectMenuBuilder()
      .setCustomId(`role_select_${i + 1}`) // 1 to 5
      .setPlaceholder(`Select your ${i + 1}${i === 0 ? "st" : i === 1 ? "nd" : i === 2 ? "rd" : "th"} preferred role`)
      .addOptions(
        ROLES.map(r => new StringSelectMenuOptionBuilder().setLabel(r.label).setValue(r.value))
      );

    rows.push({ type: 1, components: [select] }); // ActionRow type 1
  }

  await user.send({
    content: "🎮 Select your **5 preferred roles** in order (1 = most preferred, 5 = least preferred):",
    components: rows
  });
}


// ---------------- LEADERBOARD ----------------
async function updateLeaderboardChannel(guild) {
  const channelName = "leaderboard";
  let lbChannel = guild.channels.cache.find(c => c.name === channelName && c.type === 0);
  if (!lbChannel) {
    lbChannel = await guild.channels.create({ name: channelName, type: 0 });
  }

  // Build leaderboard sorted by Elo/IHP
  const players = Object.keys(playerData)
    .map(id => {
      const p = playerData[id];
      const gp = p.wins + p.losses;
      const wr = gp ? ((p.wins / gp) * 100).toFixed(1) : "0.0";
      return {
        id,
        rank: p.rank,
        division: p.division,
        lp: p.lp,
        elo: getIHP(p),
        wins: p.wins,
        losses: p.losses,
        wr,
        gp
      };
    })
    .sort((a, b) => b.elo - a.elo); // highest Elo first

  // Split into chunks of 25
  const chunkSize = 25;
  const embeds = [];
  for (let i = 0; i < players.length; i += chunkSize) {
    const chunk = players.slice(i, i + chunkSize);
    const description = chunk
      .map((p, idx) => {
        const rankDiv = p.division ? `${p.rank} ${p.division}` : p.rank;
        const lpLabel = "LP";
        const line1 = `#${i + idx + 1} • ${rankDiv} ${p.lp} LP`;
        const line2 = `<@${p.id}> | Elo: ${p.elo} | W: ${p.wins} | L: ${p.losses} | WR: ${p.wr}% | GP: ${p.gp}`;
        return `${line1}\n${line2}`;
      })
      .join("\n\n");

    const embed = new EmbedBuilder()
      .setTitle(i === 0 ? "🏆 Leaderboard" : `Leaderboard (cont.)`)
      .setDescription(description)
      .setColor(0xffd700)
      .setTimestamp();
    embeds.push(embed);
  }

  // EDIT existing messages if they exist
  if (leaderboardMessage && leaderboardMessage.length) {
    for (let i = 0; i < embeds.length; i++) {
      const embed = embeds[i];
      if (leaderboardMessage[i]) {
        // Edit existing message
        await leaderboardMessage[i].edit({ embeds: [embed] }).catch(() => {});
      } else {
        // Add new message if needed
        const msg = await lbChannel.send({ embeds: [embed] });
        leaderboardMessage.push(msg);
      }
    }
    // Delete any extra old messages
    if (leaderboardMessage.length > embeds.length) {
      for (let i = embeds.length; i < leaderboardMessage.length; i++) {
        await leaderboardMessage[i].delete().catch(() => {});
      }
      leaderboardMessage = leaderboardMessage.slice(0, embeds.length);
    }
  } else {
    // No previous messages → send new ones
    leaderboardMessage = [];
    for (const embed of embeds) {
      const msg = await lbChannel.send({ embeds: [embed] });
      leaderboardMessage.push(msg);
    }
  }
}

// ---------------- READY CHECK ----------------
async function startReadyCheck(channel) {
  const participants = [...queue];
  const ready = new Set();
  const declined = new Set();
  const TIMEOUT = 60; // seconds
  let remaining = TIMEOUT;

  const embed = new EmbedBuilder()
    .setTitle("⚔️ Ready Check")
    .setDescription(
      `${participants.length} players have queued!\n\nClick **✅ Ready** if you're ready.\nClick **❌ Decline** if you can't.\n\n⏳ Time remaining: ${remaining}s`
    )
    .setColor(0x00ffff)
    .setFooter({ text: `Waiting for players... | 0/${participants.length} ready` });

  const row = new ActionRowBuilder().addComponents(
    new ButtonBuilder()
      .setCustomId("ready")
      .setLabel("✅ Ready")
      .setStyle(ButtonStyle.Success),
    new ButtonBuilder()
      .setCustomId("notready")
      .setLabel("❌ Decline")
      .setStyle(ButtonStyle.Danger)
  );

  // Send initial message
  const msg = await channel.send({
    content: participants.map((id) => `<@${id}>`).join(" "),
    embeds: [embed],
    components: [row],
  });

  // Create collector
  const collector = msg.createMessageComponentCollector({ time: TIMEOUT * 1000 });

  // Countdown interval
  let countdown = setInterval(async () => {
    remaining--;
    if (remaining < 0) return clearInterval(countdown);

    const updatedEmbed = EmbedBuilder.from(embed)
      .setFooter({ text: `✅ Ready: ${ready.size}/${participants.length}` })
      .setDescription(
        `${participants.length} players have queued!\n\nClick **✅ Ready** if you're ready.\nClick **❌ Decline** if you can't.\n\n⏳ Time remaining: ${remaining}s`
      );

    if (declined.size > 0) {
      updatedEmbed.addFields({
        name: "❌ Declined",
        value: Array.from(declined).map((id) => `<@${id}>`).join("\n"),
      });
    }

    await msg.edit({ embeds: [updatedEmbed], components: [row] }).catch(() => {});
  }, 1000);

  // Register active ready-check so !forceready can stop it
  activeReadyCheck = { msg, collector, countdown };

  collector.on("collect", async (i) => {
    if (!participants.includes(i.user.id)) {
      return i.reply({ content: "You're not in this queue.", ephemeral: true });
    }

    if (i.customId === "notready") {
      // Remove player from queue
      queue = queue.filter((id) => id !== i.user.id);
      declined.add(i.user.id);
      saveData();
      await updateQueueMessage();

      await i.reply({
        content: "❌ You have declined the queue. You have been removed from the queue.",
        ephemeral: true,
      }).catch(() => {});

      // Update embed immediately
      const updatedEmbed = EmbedBuilder.from(embed)
        .setDescription(
          `${participants.length} players have queued!\n\nClick **✅ Ready** if you're ready.\nClick **❌ Decline** if you can't.\n\n⏳ Time remaining: ${remaining}s`
        )
        .setFooter({ text: `✅ Ready: ${ready.size}/${participants.length}` });

      if (declined.size > 0) {
        updatedEmbed.addFields({
          name: "❌ Declined",
          value: Array.from(declined).map((id) => `<@${id}>`).join("\n"),
        });
      }

      await msg.edit({ embeds: [updatedEmbed], components: [row] }).catch(() => {});
      collector.stop("declined");
      return;
    }

    // Mark ready
    ready.add(i.user.id);
    await i.deferUpdate().catch((err) => {
      if (err.code !== 10062) console.error("Button deferUpdate error:", err);
    });

    const updatedEmbed = EmbedBuilder.from(embed)
      .setDescription(
        `${participants.length} players have queued!\n\nClick **✅ Ready** if you're ready.\nClick **❌ Decline** if you can't.\n\n⏳ Time remaining: ${remaining}s`
      )
      .setFooter({ text: `✅ Ready: ${ready.size}/${participants.length}` });

    if (declined.size > 0) {
      updatedEmbed.addFields({
        name: "❌ Declined",
        value: Array.from(declined).map((id) => `<@${id}>`).join("\n"),
      });
    }

    await msg.edit({ embeds: [updatedEmbed], components: [row] }).catch(() => {});

    if (ready.size === participants.length) {
      collector.stop("all_ready");
    }
  });

  collector.on("end", async (_, reason) => {
    clearInterval(countdown);

    const finalEmbed = EmbedBuilder.from(embed)
      .setDescription(
        reason === "all_ready"
          ? "✅ All players ready. Match is starting!"
          : reason === "declined"
          ? "❌ A player declined the ready check. Match canceled."
          : "⌛ Ready check timed out. Match canceled."
      )
      .setColor(
        reason === "all_ready"
          ? 0x00ff00
          : reason === "declined"
          ? 0xff0000
          : 0xffa500
      );

    if (declined.size > 0) {
      finalEmbed.addFields({
        name: "❌ Declined",
        value: Array.from(declined).map((id) => `<@${id}>`).join("\n"),
      });
    }

    await msg.edit({ embeds: [finalEmbed], components: [] }).catch(() => {});

    if (reason === "all_ready") {
      await makeTeams(channel);
    } else if (reason === "declined") {
      await updateQueueMessage();
    } else {
      // Timeout: remove unready players
      const notReady = participants.filter((id) => !ready.has(id));
      if (notReady.length > 0) {
        queue = queue.filter((id) => !notReady.includes(id));
        saveData();
        await updateQueueMessage();
        await msg.channel.send(
          `⌛ Ready check timed out. The following players did not respond and were removed:\n${notReady
            .map((id) => `<@${id}>`)
            .join(", ")}`
        );
      }
    }

    // Clear active ready-check handle
    activeReadyCheck = null;
  });
}

async function createDraftLolLobby() {
  const browser = await puppeteer.launch({ headless: true });
  const page = await browser.newPage();
  await page.goto("https://draftlol.dawe.gg/");
  // Click "Create Lobby"
  await page.waitForSelector("div.sendButton");
  await page.click("div.sendButton");
  // Wait for blue and red inputs
  await page.waitForSelector(".createContainer input.inputBlue");
  await page.waitForSelector(".createContainer input.inputRed");
  // Wait for spectator input (any input that is not blue or red)
  await page.waitForFunction(() => {
    const container = document.querySelector(".createContainer");
    if (!container) return false;
    const inputs = Array.from(container.querySelectorAll("input[type=text]"));
    return inputs.some((input) => !input.classList.contains("inputBlue") && !input.classList.contains("inputRed"));
  });
  // Grab all three links
  const links = await page.evaluate(() => {
    const container = document.querySelector(".createContainer");
    if (!container) return { blue: "", red: "", spectator: "" };
    const blue = container.querySelector(".inputBlue")?.value || "";
    const red = container.querySelector(".inputRed")?.value || "";
    const inputs = Array.from(container.querySelectorAll("input[type=text]"));
    const spectatorInput = inputs.find(
      (input) => !input.classList.contains("inputBlue") && !input.classList.contains("inputRed")
    );
    const spectator = spectatorInput?.value || "";
    return { blue, red, spectator };
  });
  await browser.close();
  return links;
}

// --- Mute Logic ---
const mutedUsers = new Set(); // users currently muted BY THE BOT

client.on("voiceStateUpdate", async (oldState, newState) => {
  if (!matches.current) return;
  const { team1, team2, team1VC, team2VC } = matches.current;
  const member = newState.member;
  if (!member || member.user.bot) return;

  const newVC = newState.channelId;
  const oldVC = oldState.channelId;

  try {
    // User left all channels → unmute if bot muted them
    if (!newVC) {
      if (mutedUsers.has(member.id) && member.voice.serverMute) {
        await member.voice.setMute(false, "Left VC");
        mutedUsers.delete(member.id);
      }
      return;
    }

    const isTeam1VC = newVC === team1VC.id;
    const isTeam2VC = newVC === team2VC.id;
    const allowed = (isTeam1VC && team1.includes(member.id)) || (isTeam2VC && team2.includes(member.id));

    if (isTeam1VC || isTeam2VC) {
      if (!allowed) {
        // Only mute if bot hasn't already muted them manually
        if (!mutedUsers.has(member.id) && !member.voice.serverMute) {
          await member.voice.setMute(true, "Joined wrong team VC");
          mutedUsers.add(member.id);
        }
      } else {
        // Only unmute if bot muted them previously
        if (mutedUsers.has(member.id)) {
          await member.voice.setMute(false, "Joined correct team VC");
          mutedUsers.delete(member.id);
        }
      }
    } else {
      // Outside match VC → unmute if bot muted them
      if (mutedUsers.has(member.id) && member.voice.serverMute) {
        await member.voice.setMute(false, "Outside match VC");
        mutedUsers.delete(member.id);
      }
    }
  } catch (err) {
    console.error("voiceStateUpdate error:", err);
  }
});

// Reset between matches
function resetMutedUsers() {
  mutedUsers.clear();
}

function buildQueueEmbed() {
  const embed = new EmbedBuilder()
    .setTitle("🎮 Current Queue")
    .setColor(0x00ff00)
    .setDescription(
      queue.length ? queue.map((id, i) => `${i + 1}. <@${id}>`).join("\n") : "The queue is currently empty."
    )
    .setFooter({ text: `Queue Size: ${queue.length}/${QUEUE_SIZE}` })
    .setTimestamp();
  return embed;
}

// ---------------- QUEUE EMBED ----------------
async function postQueueMessage(channel) {
  const row = new ActionRowBuilder().addComponents(
    new ButtonBuilder().setCustomId("join").setLabel("✅ Join Queue").setStyle(ButtonStyle.Success),
    new ButtonBuilder().setCustomId("leave").setLabel("🚪 Leave Queue").setStyle(ButtonStyle.Danger),
    new ButtonBuilder().setCustomId("opgg").setLabel("🌐 Multi OP.GG").setStyle(ButtonStyle.Primary) // blue button
  );

  const embed = buildQueueEmbed();
  queueMessage = await channel.send({ embeds: [embed], components: [row] });
}

async function updateQueueMessage() {
  if (!queueMessage) return;

  // dynamically rebuild the Multi OP.GG link
  const getMultiOPGG = () => {
    const summoners = queue
      .map((id) => playerData[id]?.summonerName)
      .filter(Boolean)
      .map((url) => {
        try {
          const parts = url.split("/");
          const namePart = decodeURIComponent(parts[parts.length - 1]);
          return namePart.replace("-", "%23").replace(/\s+/g, "+");
        } catch {
          return null;
        }
      })
      .filter(Boolean);
    if (summoners.length === 0) return "https://www.op.gg/";
    return `https://www.op.gg/lol/multisearch/na?summoners=${summoners.join("%2C")}`;
  };

  const row = new ActionRowBuilder().addComponents(
    new ButtonBuilder().setCustomId("join").setLabel("✅ Join Queue").setStyle(ButtonStyle.Success),
    new ButtonBuilder().setCustomId("leave").setLabel("🚪 Leave Queue").setStyle(ButtonStyle.Danger),
    new ButtonBuilder().setLabel("🌐 Multi OP.GG").setStyle(ButtonStyle.Link).setURL(getMultiOPGG())
  );

  const embed = buildQueueEmbed();
  await queueMessage.edit({ embeds: [embed], components: [row] });
}

client.on("interactionCreate", async (interaction) => {
  const id = interaction.user.id;
  ensurePlayer(id);

  // ---------------- BUTTONS ----------------
  if (interaction.isButton()) {
    // --- Join Queue ---
    if (interaction.customId === "join") {
      const player = playerData[id];
      if (!player || !player.summonerName) {
        return interaction.reply({
          content: "❌ You must **register** first with !register <OP.GG link> before joining the queue.",
          ephemeral: true,
        });
      }
      if (queue.includes(id)) {
        return interaction.reply({ content: "⚠️ You're already in the queue.", ephemeral: true });
      }
      queue.push(id);
      saveData();
      await updateQueueMessage();
      if (queue.length >= QUEUE_SIZE) await startReadyCheck(interaction.channel);
      return interaction.deferUpdate();
    }

    // --- Leave Queue ---
    if (interaction.customId === "leave") {
      if (!queue.includes(id)) return interaction.deferUpdate(); // silently ignore
      queue = queue.filter((x) => x !== id);
      saveData();
      await updateQueueMessage();
      return interaction.deferUpdate();
    }

    // --- Multi OP.GG ---
    if (interaction.customId === "opgg") {
      if (queue.length === 0) {
        return interaction.reply({ content: "❌ The queue is empty.", ephemeral: true });
      }
      const summoners = queue
        .map((id) => playerData[id]?.summonerName)
        .filter(Boolean)
        .map((url) => {
          try {
            const parts = url.split("/");
            const namePart = decodeURIComponent(parts[parts.length - 1]);
            return namePart.replace("-", "%23").replace(/\s+/g, "+");
          } catch {
            return null;
          }
        })
        .filter(Boolean);

      if (summoners.length === 0) {
        return interaction.reply({ content: "❌ No registered OP.GG links found.", ephemeral: true });
      }

      const multiLink = `https://www.op.gg/lol/multisearch/na?summoners=${summoners.join("%2C")}`;
      return interaction.reply({ content: `🌐 [Multi OP.GG Link for Queue](${multiLink})`, ephemeral: true });
    }

    // --- End Match ---
    if (interaction.customId === "endmatch") {
      return endMatch(interaction.channel, "manual");
    }
  }

  // ---------------- SELECT MENUS ----------------
  else if (interaction.isStringSelectMenu()) {
    const userId = interaction.user.id;
    ensurePlayer(userId);

    if (!playerData[userId].roles) playerData[userId].roles = [];

    // parse the slot number from customId: role_select_<userId>_<1-5>
    const parts = interaction.customId.split("_");
    const slot = parseInt(parts[3]) - 1; // 0-4
    playerData[userId].roles[slot] = interaction.values[0]; // store role
    saveData();

    await interaction.reply({
      content: `✅ Saved your role choice for slot #${slot + 1}: ${interaction.values[0]}`,
      ephemeral: true,
    });
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

  // ---------------- !forceready ----------------
  if (cmd === "!forceready") {
  if (!message.member.permissions.has("ManageGuild")) {
    return message.reply("❌ Only staff members can use this command.");
  }

  if (!queue || queue.length < QUEUE_SIZE) {
    return message.reply("⚠️ There isn't an active ready check right now.");
  }

  message.channel.send("✅ Force-ready activated — all players are now marked ready!");

  // If there's an active ready-check, stop its collector so the existing
  // end-handler runs (it will call makeTeams). This avoids duplicate calls.
  if (activeReadyCheck && activeReadyCheck.collector) {
    try {
      // Optionally edit the message for immediate feedback and remove buttons
      const curMsg = activeReadyCheck.msg;
      try {
        const infoEmbed = EmbedBuilder.from(curMsg.embeds?.[0] || embed)
          .setDescription("✅ Force-ready activated by staff. Match is starting!")
          .setColor(0x00ff00);
        await curMsg.edit({ embeds: [infoEmbed], components: [] }).catch(() => {});
      } catch (e) { /* ignore UI edit errors */ }

      // Stop collector — use same reason as normal "all_ready" path
      activeReadyCheck.collector.stop("all_ready");
      // don't call makeTeams() here — collector.on('end') will do it.
    } catch (err) {
      console.error("Error stopping active ready check:", err);
      // fallback: if something went wrong and no collector, call makeTeams
      await makeTeams(message.channel);
    }
  } else {
    // No active ready-check object (maybe message not cached) — fallback:
    await makeTeams(message.channel);
  }
}

  if (cmd === "!remove") {
    if (!message.member.permissions.has("ManageGuild")) {
      return message.reply("❌ You need Manage Server permissions to do that.");
    }
    const userMention = args[0];
    if (!userMention) return message.reply("Usage: !removefromqueue <@user>");
    const userId = userMention.replace(/[<@!>]/g, "");
    if (!queue.includes(userId)) {
      return message.reply("⚠️ That user is not in the queue.");
    }
    queue = queue.filter((id) => id !== userId);
    saveData();
    await updateQueueMessage();
    return message.channel.send(`🚫 Removed <@${userId}> from the queue.`);
  }

  if (cmd === "!cancelmatch") {
  // Only allow staff (ManageGuild permission) to run this command
  if (!message.member.permissions.has("ManageGuild")) {
    return message.reply("❌ Only staff members can use this command.");
  }

  // If no active match exists
  if (!matches.current) {
    return message.reply("⚠️ There is no active match to cancel.");
  }

  const { matchChannel, team1VC, team2VC } = matches.current;
  const guild = message.guild;

  // Use #general as the safe channel for confirmation
  const safeChannel = guild.channels.cache.find(c => c.name === "general" && c.type === 0) || message.channel;

  try {
    // Delete all match-related channels if they still exist
    if (matchChannel) await matchChannel.delete().catch(() => {});
    if (team1VC) await team1VC.delete().catch(() => {});
    if (team2VC) await team2VC.delete().catch(() => {});

    // Optionally delete the match category (if it exists and is empty)
    const matchCategory = guild.channels.cache.find(c => c.type === 4 && c.name.toLowerCase() === "match");
    if (matchCategory) await matchCategory.delete().catch(() => {});

    // Reset active match reference
    matches.current = null;

    // Reset mute tracking for the next match
    resetMutedUsers();

    // Send confirmation safely to general
    await safeChannel.send("🛑 The current match has been canceled and all match channels have been deleted.");
  } catch (err) {
    console.error("Error canceling match:", err);
    await safeChannel.send("⚠️ Failed to fully cancel the match. Check console for details.");
  }
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
    // Only allow staff (ManageGuild permission) to run this command
    if (!message.member.permissions.has("ManageGuild")) {
      return message.reply("❌ Only staff members can use this command.");
    }
    const team = args[0];
    if (!team || !matches.current) {
      return message.reply("Usage: !endmatch <1|2>");
    }
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
        const romanToNumber = { IV: 4, III: 3, II: 2, I: 1 };
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
      await updateLeaderboardChannel(message.guild); // or channel.guild
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
  if (!message.content.startsWith("!register")) return;

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
      const romanToNumber = { IV: 4, III: 3, II: 2, I: 1 };
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
    await updateLeaderboardChannel(message.guild);

    await message.reply(`✅ Registered ${message.author.username} as **${tierText} ${lp} LP**`);

    // ---------------- Role Selection Dropdowns ----------------
    const rows = [];
    for (let i = 0; i < 5; i++) {
      const selectMenu = new StringSelectMenuBuilder()
        .setCustomId(`role_select_${userId}_${i + 1}`) // unique per user
        .setPlaceholder(`Select your ${i + 1}${i === 0 ? "st" : i === 1 ? "nd" : i === 2 ? "rd" : "th"} preferred role`)
        .addOptions(
          ROLES.map(r => ({ label: r.label, value: r.value }))
        );
      rows.push(new ActionRowBuilder().addComponents(selectMenu));
    }

    await message.channel.send({
      content: `${message.author}, please select your **5 preferred roles** (1 = most preferred, 5 = least preferred):`,
      components: rows
    });

  } catch (err) {
    console.error(err);
    return message.reply("❌ Failed to fetch OP.GG page. Make sure the link is correct.");
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
    (c) => c.type === 4 && c.name.toLowerCase() === "match"
  );
  if (!matchCategory) {
    matchCategory = await guild.channels.create({ name: "Match", type: 4 });
  }
  const matchChannel = await guild.channels.create({ name: "match", type: 0, parent: matchCategory.id });
  const team1VC = await guild.channels.create({ name: "Team 1 (Blue Side)", type: 2, parent: matchCategory.id });
  const team2VC = await guild.channels.create({ name: "Team 2 (Red Side)", type: 2, parent: matchCategory.id });

  const embed = new EmbedBuilder()
    .setTitle("🎮 Match Ready!")
    .addFields(
      { name: `Team 1 (Avg Elo: ${avg1})`, value: bestTeam1.map((id) => formatPlayer(id)).join("\n"), inline: true },
      { name: `Team 2 (Avg Elo: ${avg2})`, value: bestTeam2.map((id) => formatPlayer(id)).join("\n"), inline: true }
    )
    .setFooter({ text: "Use the team voice channels below to join your squad!" });

  await matchChannel.send({ embeds: [embed] });

  // --- Build Multi OP.GG Links for each team ---
  const buildMulti = (team) => {
    const summoners = team
      .map((id) => playerData[id]?.summonerName)
      .filter(Boolean)
      .map((url) => {
        try {
          const parts = url.split("/");
          const namePart = decodeURIComponent(parts[parts.length - 1]);
          return namePart.replace("-", "%23").replace(/\s+/g, "+");
        } catch {
          return null;
        }
      })
      .filter(Boolean);
    if (summoners.length === 0) return "https://www.op.gg/";
    return `https://www.op.gg/lol/multisearch/na?summoners=${summoners.join("%2C")}`;
  };

  const team1Link = buildMulti(bestTeam1);
  const team2Link = buildMulti(bestTeam2);

  // --- Create Draft Lobby using draftlol.dawe.gg ---
  try {
    const { blue, red, spectator } = await createDraftLolLobby();
    await matchChannel.send(
      `🌐 **Match Links**\n + 🟦 **Blue Team OP.GG:** <${team1Link}>\n + 🟥 **Red Team OP.GG:** <${team2Link}>\n + 🎯 **Draft Lobby:**\n + Blue: <${blue}>\n + Red: <${red}>\n + Spectator: <${spectator}>`
    );
  } catch (err) {
    console.error("Failed to create draft lobby:", err);
    await matchChannel.send(
      `🌐 **Match Links**\n🟦 **Blue Team OP.GG:** <${team1Link}>\n🟥 **Red Team OP.GG:** <${team2Link}>\n❌ Failed to create Draft Lobby. Players will need to pick manually.`
    );
  }

  matches.current = { team1: bestTeam1, team2: bestTeam2, matchChannel, team1VC, team2VC };
  queue = [];
  saveData();
  updateQueueMessage();

  // Immediately apply mute logic to all players in voice channels
  for (const member of channel.guild.members.cache.values()) {
    const state = member.voice;
    if (!state?.channelId) continue;
    client.emit("voiceStateUpdate", { channelId: null, member }, state);
  }
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

  // Fun shoutout if Romeo wins
  if (winners.includes("272603932268822529")) {
    if (general) general.send("Wow Romeo actually won? That guy is ass");
  }

  // Update winners
  winners.forEach((id) => {
    const p = ensurePlayer(id);
    const oldIHP = getIHP(p);
    const oldRank = p.rank;
    const oldDivision = p.division;

    p.wins++;
    p.lp += 20;

    checkRankChange(id, general, p, oldIHP, oldRank, oldDivision);
  });

  // Update losers
  losers.forEach((id) => {
    const p = ensurePlayer(id);
    const oldIHP = getIHP(p);
    const oldRank = p.rank;
    const oldDivision = p.division;

    p.losses++;
    p.lp -= 20;

    checkRankChange(id, general, p, oldIHP, oldRank, oldDivision);
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

  // Delete match channels
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
  resetMutedUsers(); // Reset mute tracking for next match
}

async function checkRankChange(id, announceChannel, player, oldIHP, oldRank, oldDivision) {
  const newStats = IHPToRank(getIHP(player), player); // pass current player
  Object.assign(player, newStats);

  const newIHP = getIHP(player);

  if (player.rank !== oldRank || player.division !== oldDivision) {
    if (newIHP > oldIHP) {
      await announceChannel.send(
        `🎉 <@${id}> ranked up to **${player.rank}${player.division ? " " + player.division : ""}**! 🏅`
      );
    } else if (newIHP < oldIHP) {
      await announceChannel.send(
        `⬇️ <@${id}> has been demoted to **${player.rank}${player.division ? " " + player.division : ""}**.`
      );
    }

    await updateLeaderboardChannel(announceChannel.guild);
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

  // Call leaderboard properly
  await updateLeaderboardChannel(guild);
});

client.login(process.env.BOT_TOKEN);
