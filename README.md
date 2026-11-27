
const {
  Client,
  GatewayIntentBits,
  ChannelType,
  PermissionsBitField,
  EmbedBuilder,
  ActionRowBuilder,
  ButtonBuilder,
  ButtonStyle,
  SlashCommandBuilder,
  REST,
  Routes,
  AttachmentBuilder
} = require("discord.js");

// ==================================
// CONFIGURATION
// ==================================

const TOKEN = "YOUR_BOT_TOKEN";
const CLIENT_ID = "YOUR_CLIENT_ID";
const GUILD_ID = "YOUR_SERVER_ID";

const CONFIG = {
  CATEGORY_ID: "YOUR_TICKET_CATEGORY_ID",
  LOG_CHANNEL_ID: "YOUR_LOG_CHANNEL_ID",
  STAFF_ROLE_ID: "YOUR_STAFF_ROLE_ID"
};

// ==================================
// TICKET TYPES
// ==================================

const TICKETS = {
  support: {
    label: "الدعم الفني",
    emoji: "🛠️",
    style: ButtonStyle.Primary
  },

  admin: {
    label: "تقديم الإدارة",
    emoji: "👮",
    style: ButtonStyle.Success
  },

  complaint: {
    label: "الشكاوى",
    emoji: "📩",
    style: ButtonStyle.Danger
  },

  partnership: {
    label: "الشراكات",
    emoji: "🤝",
    style: ButtonStyle.Secondary
  },

  general: {
    label: "استفسار عام",
    emoji: "❓",
    style: ButtonStyle.Secondary
  }
};

// ==================================
// CLIENT
// ==================================

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent
  ]
});

// ==================================
// SLASH COMMANDS
// ==================================

const commands = [
  new SlashCommandBuilder()
    .setName("setup")
    .setDescription("إنشاء لوحة التكت")
    .toJSON()
];

const rest = new REST({ version: "10" }).setToken(TOKEN);

async function registerCommands() {
  await rest.put(
    Routes.applicationGuildCommands(CLIENT_ID, GUILD_ID),
    { body: commands }
  );

  console.log("Slash commands registered");
}

// ==================================
// HELPERS
// ==================================

function isAdmin(interaction) {
  return interaction.memberPermissions?.has(
    PermissionsBitField.Flags.Administrator
  );
}

function isStaff(interaction) {
  return (
    isAdmin(interaction) ||
    interaction.member?.roles?.cache?.has(CONFIG.STAFF_ROLE_ID)
  );
}

async function sendLog(guild, embed, file = null) {
  const logChannel = await guild.channels.fetch(
    CONFIG.LOG_CHANNEL_ID
  ).catch(() => null);

  if (!logChannel || !logChannel.isTextBased()) {
    console.error("Log channel not found");
    return false;
  }

  await logChannel.send({
    embeds: [embed],
    files: file ? [file] : []
  });

  return true;
}

function ticketInfo(channel) {
  const parts = channel.topic?.split("|") || [];

  if (parts[0] !== "ticket") return null;

  return {
    ownerId: parts[1],
    type: parts[2],
    claimedBy: parts[3] || ""
  };
}

function ticketName(username) {
  const safe = username
    .toLowerCase()
    .replace(/[^a-z0-9]/g, "")
    .slice(0, 15);

  return `ticket-${safe || "member"}`;
}

// ==================================
// READY
// ==================================

client.once("ready", () => {
  console.log("--------------------------------");
  console.log(`BOT ONLINE: ${client.user.tag}`);
  console.log("TICKET SYSTEM V2");
  console.log("--------------------------------");
});

// ==================================
// INTERACTIONS
// ==================================

client.on("interactionCreate", async interaction => {

  try {

    // ==================================
    // /SETUP
    // ==================================

    if (interaction.isChatInputCommand()) {

      if (interaction.commandName !== "setup") return;

      if (!isAdmin(interaction)) {
        return interaction.reply({
          content: "❌ هذا الأمر للإدارة فقط.",
          ephemeral: true
        });
      }

      const embed = new EmbedBuilder()
        .setColor("#5865F2")
        .setTitle("🎫 مركز التذاكر")
        .setDescription(
          "مرحباً بك في نظام التذاكر.\n\n" +
          "اختر القسم المناسب من الأزرار بالأسفل، " +
          "وسيتم فتح روم خاص بك."
        )
        .setFooter({
          text: "RP Ticket System V2"
        })
        .setTimestamp();

      const row = new ActionRowBuilder();

      for (const [id, ticket] of Object.entries(TICKETS)) {

        row.addComponents(
          new ButtonBuilder()
            .setCustomId(`open_${id}`)
            .setLabel(ticket.label)
            .setEmoji(ticket.emoji)
            .setStyle(ticket.style)
        );

      }

      await interaction.reply({
        embeds: [embed],
        components: [row]
      });

      return;
    }

    if (!interaction.isButton()) return;

    // ==================================
    // OPEN TICKET
    // ==================================

    if (interaction.customId.startsWith("open_")) {

      const type = interaction.customId.replace("open_", "");
      const ticket = TICKETS[type];

      if (!ticket) return;

      const existing = interaction.guild.channels.cache.find(
        channel =>
          channel.type === ChannelType.GuildText &&
          ticketInfo(channel)?.ownerId === interaction.user.id
      );

      if (existing) {
        return interaction.reply({
          content: `❌ لديك تكت مفتوح بالفعل: ${existing}`,
          ephemeral: true
        });
      }

      await interaction.deferReply({
        ephemeral: true
      });

      const channel = await interaction.guild.channels.create({

        name: ticketName(interaction.user.username),

        type: ChannelType.GuildText,

        parent: CONFIG.CATEGORY_ID,

        topic: `ticket|${interaction.user.id}|${type}|`,

        permissionOverwrites: [

          {
            id: interaction.guild.id,

            deny: [
              PermissionsBitField.Flags.ViewChannel
            ]
          },

          {
            id: interaction.user.id,

            allow: [
              PermissionsBitField.Flags.ViewChannel,
              PermissionsBitField.Flags.SendMessages,
              PermissionsBitField.Flags.ReadMessageHistory,
              PermissionsBitField.Flags.AttachFiles
            ]
          },

          {
            id: CONFIG.STAFF_ROLE_ID,

            allow: [
              PermissionsBitField.Flags.ViewChannel,
              PermissionsBitField.Flags.SendMessages,
              PermissionsBitField.Flags.ReadMessageHistory
            ]
          },

          {
            id: client.user.id,

            allow: [
              PermissionsBitField.Flags.ViewChannel,
              PermissionsBitField.Flags.SendMessages,
              PermissionsBitField.Flags.ReadMessageHistory,
              PermissionsBitField.Flags.ManageChannels,
              PermissionsBitField.Flags.AttachFiles
            ]
          }

        ]

      });

      // TICKET EMBED

      const ticketEmbed = new EmbedBuilder()
        .setColor("#2B2D31")
        .setTitle(`🎫 ${ticket.label}`)
        .setDescription(
          `أهلاً بك ${interaction.user}\n\n` +
          "يرجى كتابة طلبك بالتفصيل.\n" +
          "سيتم الرد عليك من قبل الإدارة."
        )
        .addFields(
          {
            name: "👤 صاحب التكت",
            value: `${interaction.user}`
          },
          {
            name: "📂 القسم",
            value: ticket.label
          },
          {
            name: "📌 الحالة",
            value: "بانتظار استلام الإدارة"
          }
        )
        .setTimestamp();

      // CLAIM AND CLOSE BUTTONS

      const buttons = new ActionRowBuilder()
        .addComponents(

          new ButtonBuilder()
            .setCustomId("claim_ticket")
            .setLabel("استلام التكت")
            .setEmoji("📥")
            .setStyle(ButtonStyle.Success),

          new ButtonBuilder()
            .setCustomId("close_ticket")
            .setLabel("إغلاق التكت")
            .setEmoji("🔒")
            .setStyle(ButtonStyle.Danger)

        );

      await channel.send({

        content:
          `${interaction.user} <@&${CONFIG.STAFF_ROLE_ID}>`,

        embeds: [ticketEmbed],

        components: [buttons],

        allowedMentions: {
          users: [interaction.user.id],
          roles: [CONFIG.STAFF_ROLE_ID]
        }

      });

      // OPEN LOG

      const log = new EmbedBuilder()
        .setColor("#57F287")
        .setTitle("📂 فتح تكت جديد")
        .addFields(
          {
            name: "صاحب التكت",
            value: `${interaction.user}`
          },
          {
            name: "نوع التكت",
            value: ticket.label
          },
          {
            name: "روم التكت",
            value: `${channel}`
          }
        )
        .setTimestamp();

      await sendLog(interaction.guild, log);

      await interaction.editReply({
        content: `✅ تم فتح التكت بنجاح: ${channel}`
      });

      return;
    }

    // ==================================
    // CLAIM TICKET
    // ==================================

    if (interaction.customId === "claim_ticket") {

      if (!isStaff(interaction)) {
        return interaction.reply({
          content: "❌ هذا الزر للإدارة فقط.",
          ephemeral: true
        });
      }

      const info = ticketInfo(interaction.channel);

      if (!info) {
        return interaction.reply({
          content: "هذا الروم ليس تكت.",
          ephemeral: true
        });
      }

      if (info.claimedBy) {
        return interaction.reply({
          content: `❌ التكت مستلم بالفعل من <@${info.claimedBy}>.`,
          ephemeral: true
        });
      }

      await interaction.channel.setTopic(
        `ticket|${info.ownerId}|${info.type}|${interaction.user.id}`
      );

      const claimedEmbed = new EmbedBuilder()
        .setColor("#57F287")
        .setTitle("📥 تم استلام التكت")
        .setDescription(
          `تم استلام التكت بواسطة ${interaction.user}`
        )
        .setTimestamp();

      await interaction.channel.send({
        embeds: [claimedEmbed]
      });

      const claimLog = new EmbedBuilder()
        .setColor("#FEE75C")
        .setTitle("📥 استلام تكت")
        .addFields(
          {
            name: "صاحب التكت",
            value: `<@${info.ownerId}>`
          },
          {
            name: "الإداري المستلم",
            value: `${interaction.user}`
          },
          {
            name: "الروم",
            value: `${interaction.channel}`
          }
        )
        .setTimestamp();

      await sendLog(interaction.guild, claimLog);

      await interaction.reply({
        content: "✅ تم استلام التكت بنجاح."
      });

      return;
    }

    // ==================================
    // CLOSE TICKET
    // ==================================

    if (interaction.customId === "close_ticket") {

      const channel = interaction.channel;
      const info = ticketInfo(channel);

      if (!info) {
        return interaction.reply({
          content: "❌ هذا الروم ليس تكت.",
          ephemeral: true
        });
      }

      const owner = interaction.user.id === info.ownerId;

      if (!owner && !isStaff(interaction)) {
        return interaction.reply({
          content: "❌ ليس لديك صلاحية إغلاق التكت.",
          ephemeral: true
        });
      }

      await interaction.deferReply({
        ephemeral: true
      });

      // FETCH ALL MESSAGES

      let messages = [];
      let lastId;

      while (true) {

        const batch = await channel.messages.fetch({
          limit: 100,
          ...(lastId ? { before: lastId } : {})
        });

        if (batch.size === 0) break;

        messages.push(...batch.values());

        lastId = batch.last().id;

        if (batch.size < 100) break;

      }

      // SORT MESSAGES

      messages.sort(
        (a, b) => a.createdTimestamp - b.createdTimestamp
      );

      // BUILD TRANSCRIPT

      const transcript = messages.map(message => {

        const date = message.createdAt.toISOString();

        const attachments = [
          ...message.attachments.values()
        ].map(file => file.url).join(", ");

        const content = message.content || "[No text content]";

        const embeds = message.embeds.length
          ? "[Message contains embed]"
          : "";

        return (
          `[${date}] ${message.author.tag}\n` +
          `${content}\n` +
          `${attachments}\n` +
          `${embeds}\n` +
          "----------------------------------------"
        );

      }).join("\n");

      const file = new AttachmentBuilder(
        Buffer.from(
          transcript || "No messages found.",
          "utf8"
        ),
        {
          name: `ticket-${channel.id}-transcript.txt`
        }
      );

      // CLOSE LOG

      const closeLog = new EmbedBuilder()
        .setColor("#ED4245")
        .setTitle("🔒 إغلاق تكت")
        .addFields(
          {
            name: "صاحب التكت",
            value: `<@${info.ownerId}>`
          },
          {
            name: "أغلق بواسطة",
            value: `${interaction.user}`
          },
          {
            name: "الإداري المستلم",
            value: info.claimedBy
              ? `<@${info.claimedBy}>`
              : "لم يتم الاستلام"
          },
          {
            name: "نوع التكت",
            value: TICKETS[info.type]?.label || info.type
          },
          {
            name: "الروم",
            value: channel.name
          }
        )
        .setTimestamp();

      // SEND TRANSCRIPT TO LOG

      await sendLog(
        interaction.guild,
        closeLog,
        file
      );

      await interaction.editReply({
        content: "✅ تم حفظ المحادثة وإغلاق التكت."
      });

      // DELETE CHANNEL

      await channel.delete("Ticket closed by user or staff");

      return;
    }

  } catch (error) {

    console.error("TICKET ERROR:", error);

    if (
      interaction.isRepliable() &&
      !interaction.replied &&
      !interaction.deferred
    ) {
      await interaction.reply({
        content: "❌ حدث خطأ أثناء تنفيذ العملية.",
        ephemeral: true
      }).catch(() => {});
    }

  }

});

// ==================================
// START BOT
// ==================================

async function start() {

  if (
    TOKEN === "YOUR_BOT_TOKEN" ||
    CLIENT_ID === "YOUR_CLIENT_ID" ||
    GUILD_ID === "YOUR_SERVER_ID"
  ) {
    throw new Error("Please configure your bot IDs and token.");
  }

  await registerCommands();

  await client.login(TOKEN);

}

start().catch(console.error);
