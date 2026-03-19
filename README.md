import discord
from discord import app_commands
from discord.ext import commands
import sqlite3
import time

from flask import Flask
from threading import Thread

app = Flask('')

@app.route('/')
def home():
    return "Bot is running!"

def run():
    app.run(host='0.0.0.0', port=8080)

Thread(target=run).start()
# ---------------- CONFIG ----------------
import os
TOKEN = os.environ['TOKEN']
OWNER_ID = 1185991173958008843

LOG_CHANNEL_ID = 1484005383612924087       # Scam logs
VOUCH_CHANNEL_ID = 1447619830269349983     # Vouch display channel
TRANSCRIPT_CHANNEL_ID = 1484005451086561331  # Transcript storage

intents = discord.Intents.all()
bot = commands.Bot(command_prefix="!", intents=intents)

# ---------------- DATABASE ----------------
db = sqlite3.connect("valor_mmhub.db")
cursor = db.cursor()

# Ratings & Vouches
cursor.execute("""CREATE TABLE IF NOT EXISTS ratings (user_id INTEGER PRIMARY KEY, rating INTEGER)""")
cursor.execute("""CREATE TABLE IF NOT EXISTS vouches (user_id INTEGER, reviewer_id INTEGER, message TEXT)""")

# Economy
cursor.execute("""CREATE TABLE IF NOT EXISTS economy (user_id INTEGER PRIMARY KEY, tokens INTEGER, messages INTEGER, last_daily INTEGER)""")

db.commit()

active_tickets = {}  # user_id: channel_id
cooldowns = {}       # user_id: timestamp

# ---------------- EMBED UTILITY ----------------
def embed(title, desc):
    return discord.Embed(title=title, description=desc, color=0x7d3cff)

def progress_bar(current, goal, length=10):
    ratio = current / goal
    filled = int(ratio * length)
    return "🟣" * filled + "⚫" * (length - filled)

# ---------------- MODALS ----------------
class RequestMMModal(discord.ui.Modal, title="💰 Request a Middleman"):
    deal = discord.ui.TextInput(label="What is the deal for?", required=True)
    size = discord.ui.TextInput(label="Short or Long deal?", required=True)
    payment = discord.ui.TextInput(label="PayPal or CashApp?", required=True)

    async def on_submit(self, interaction: discord.Interaction):
        user = interaction.user
        if user.id in active_tickets:
            return await interaction.response.send_message("❌ You already have a ticket.", ephemeral=True)
        overwrites = {
            interaction.guild.default_role: discord.PermissionOverwrite(read_messages=False),
            user: discord.PermissionOverwrite(read_messages=True, send_messages=True),
            interaction.guild.me: discord.PermissionOverwrite(read_messages=True)
        }
        channel = await interaction.guild.create_text_channel(f"mm-{user.name}", overwrites=overwrites)
        active_tickets[user.id] = channel.id
        e = embed("🧾 Middleman Ticket",
                  f"**User:** {user.mention}\n**Deal:** {self.deal.value}\n**Type:** {self.size.value}\n**Payment:** {self.payment.value}")
        await channel.send(f"{user.mention} <@{OWNER_ID}>", embed=e)
        await interaction.response.send_message(f"✅ Ticket created: {channel.mention}", ephemeral=True)

class ApplyMMModal(discord.ui.Modal, title="🛡️ Apply to Become MM"):
    reason = discord.ui.TextInput(label="Why do you want to be a MM?", required=True)
    vouches = discord.ui.TextInput(label="How many vouches? (Min 75)", required=True)
    server = discord.ui.TextInput(label="Do you own a server?", required=True)

    async def on_submit(self, interaction: discord.Interaction):
        user = interaction.user
        if user.id in active_tickets:
            return await interaction.response.send_message("❌ You already have a ticket.", ephemeral=True)
        overwrites = {
            interaction.guild.default_role: discord.PermissionOverwrite(read_messages=False),
            user: discord.PermissionOverwrite(read_messages=True, send_messages=True),
            interaction.guild.me: discord.PermissionOverwrite(read_messages=True)
        }
        channel = await interaction.guild.create_text_channel(f"mm-application-{user.name}", overwrites=overwrites)
        active_tickets[user.id] = channel.id
        e = embed("🛡️ MM Application Ticket",
                  f"**User:** {user.mention}\n**Reason:** {self.reason.value}\n**Vouches:** {self.vouches.value}\n**Own Server:** {self.server.value}")
        await channel.send(f"<@{OWNER_ID}>", embed=e)
        await interaction.response.send_message(f"✅ Application ticket created: {channel.mention}", ephemeral=True)

# ---------------- TICKET BUTTONS ----------------
class TicketButtons(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)

    @discord.ui.button(label="💰 Request a Middleman", style=discord.ButtonStyle.green)
    async def request_mm(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.send_modal(RequestMMModal())

    @discord.ui.button(label="🛡️ Apply to Become MM", style=discord.ButtonStyle.blurple)
    async def apply_mm(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.send_modal(ApplyMMModal())

# ---------------- TICKET PANEL ----------------
@bot.tree.command(name="ticketpanel", description="Post the MM ticket panel")
async def ticketpanel(interaction: discord.Interaction):
    if interaction.user.id != OWNER_ID:
        return await interaction.response.send_message("❌ You're not The man, The myth, The legend Valor?", ephemeral=True)
    e = embed("🎟️ Valor’s Exchange", "Choose an option below to create a ticket:")
    await interaction.response.send_message(embed=e, view=TicketButtons())

# ---------------- VOUCH SYSTEM ----------------
@bot.tree.command(name="vouchmm", description="Vouch for a middleman")
@app_commands.describe(user="Middleman to vouch for", message="Your vouch message")
async def vouchmm(interaction: discord.Interaction, user: discord.Member, message: str):
    if user.id == interaction.user.id:
        return await interaction.response.send_message("❌ You cannot vouch for yourself!", ephemeral=True)
    cursor.execute("SELECT * FROM vouches WHERE user_id=? AND reviewer_id=?", (user.id, interaction.user.id))
    if cursor.fetchone():
        return await interaction.response.send_message("❌ You have already vouched for this user!", ephemeral=True)
    cursor.execute("INSERT INTO vouches (user_id, reviewer_id, message) VALUES (?, ?, ?)", (user.id, interaction.user.id, message))
    cursor.execute("SELECT COUNT(*) FROM vouches WHERE user_id=?", (user.id,))
    total_vouches = cursor.fetchone()[0]
    cursor.execute("INSERT OR REPLACE INTO ratings (user_id, rating) VALUES (?, ?)", (user.id, total_vouches))
    db.commit()
    await interaction.response.send_message(f"⭐ {user.mention} now has **{total_vouches} vouches!**", ephemeral=False)

# ---------------- STATS ----------------
class ReviewButton(discord.ui.View):
    def __init__(self, user_id):
        super().__init__(timeout=None)
        self.user_id = user_id

    @discord.ui.button(label="📋 Reviews", style=discord.ButtonStyle.blurple)
    async def reviews(self, interaction: discord.Interaction, button: discord.ui.Button):
        cursor.execute("SELECT reviewer_id, message FROM vouches WHERE user_id=?", (self.user_id,))
        data = cursor.fetchall()
        if not data:
            return await interaction.response.send_message("No reviews yet.", ephemeral=True)
        text = ""
        for reviewer_id, message in data:
            reviewer = await bot.fetch_user(reviewer_id)
            text += f"**{reviewer.name}**: {message}\n"
        await interaction.response.send_message(embed=embed("📋 Reviews", text), ephemeral=True)

@bot.tree.command(name="stats", description="Check MM stats")
@app_commands.describe(user="Member to check stats")
async def stats(interaction: discord.Interaction, user: discord.Member):
    cursor.execute("SELECT rating FROM ratings WHERE user_id=?", (user.id,))
    data = cursor.fetchone()
    rating = data[0] if data else 0
    if user.id == OWNER_ID:
        tier = "Owner"
    elif rating >= 100:
        tier = "Godly Trusted"
    elif rating >= 50:
        tier = "Expert Trusted"
    elif rating >= 25:
        tier = "Lil More Trusted"
    elif rating >= 10:
        tier = "Trusted"
    else:
        tier = "Member"
    await interaction.response.send_message(embed=embed("📊 MM Stats",
                                                       f"User: {user.mention}\nVouches: {rating}\nTier: {tier}"),
                                          view=ReviewButton(user.id))

# ---------------- LEADERBOARD ----------------
@bot.tree.command(name="leaderboard", description="Show top MMs")
async def leaderboard(interaction: discord.Interaction):
    cursor.execute("SELECT user_id, rating FROM ratings ORDER BY rating DESC LIMIT 10")
    data = cursor.fetchall()
    text = ""
    for i, r in enumerate(data, 1):
        user = await bot.fetch_user(r[0])
        text += f"{i}. {user.name}: {r[1]} vouches\n"
    await interaction.response.send_message(embed=embed("🏆 MM Leaderboard", text))

# ---------------- REMOVE VOUCH ----------------
class RemoveVouchSelect(discord.ui.Select):
    def __init__(self, options, user_id):
        super().__init__(placeholder="Select a vouch to remove...", min_values=1, max_values=1, options=options)
        self.user_id = user_id

    async def callback(self, interaction: discord.Interaction):
        rowid_to_remove = int(self.values[0])
        cursor.execute("DELETE FROM vouches WHERE rowid=?", (rowid_to_remove,))
        cursor.execute("SELECT COUNT(*) FROM vouches WHERE user_id=?", (self.user_id,))
        new_rating = cursor.fetchone()[0]
        cursor.execute("INSERT OR REPLACE INTO ratings (user_id, rating) VALUES (?, ?)", (self.user_id, new_rating))
        db.commit()
        await interaction.response.send_message(f"✅ Removed selected vouch. New total: {new_rating}", ephemeral=True)

class RemoveVouchView(discord.ui.View):
    def __init__(self, user_id, interaction):
        super().__init__(timeout=60)
        self.user_id = user_id
        cursor.execute("SELECT rowid, reviewer_id, message FROM vouches WHERE user_id=?", (user_id,))
        vouches = cursor.fetchall()
        if not vouches:
            return
        options = []
        for row in vouches:
            reviewer_id = row[1]
            reviewer = interaction.guild.get_member(reviewer_id)
            reviewer_name = reviewer.display_name if reviewer else f"Unknown ({reviewer_id})"
            msg_preview = row[2][:50] + ("..." if len(row[2]) > 50 else "")
            options.append(discord.SelectOption(label=reviewer_name, description=msg_preview, value=str(row[0])))
        self.add_item(RemoveVouchSelect(options, user_id))

@bot.tree.command(name="removevouch", description="Remove a vouch from a user (owner only)")
@app_commands.describe(user="Member to remove vouches from")
async def removevouch(interaction: discord.Interaction, user: discord.Member):
    if interaction.user.id != OWNER_ID:
        return await interaction.response.send_message("❌ You're not The man, The myth, The legend Valor?", ephemeral=True)
    cursor.execute("SELECT rowid, reviewer_id, message FROM vouches WHERE user_id=?", (user.id,))
    data = cursor.fetchall()
    if not data:
        return await interaction.response.send_message(f"❌ {user.mention} has no vouches to remove.", ephemeral=True)
    view = RemoveVouchView(user.id, interaction)
    await interaction.response.send_message(f"Select a vouch to remove for {user.mention}:", view=view, ephemeral=True)

# ---------------- HELP ----------------
@bot.tree.command(name="help", description="Shows all available commands for members")
async def help_command(interaction: discord.Interaction):
    help_embed = discord.Embed(
        title="📖 Valor's Exchange - Member Commands",
        description="Here are all the commands you can use as a normal member:",
        color=0x7d3cff
    )
    help_embed.add_field(name="/vouchmm @user <message>", value="⭐ Vouch for a middleman and add to their rating.", inline=False)
    help_embed.add_field(name="/stats @user", value="📊 View a user's rating, tier, and reviews.", inline=False)
    help_embed.add_field(name="/leaderboard", value="🏆 View the top middlemen in the server.", inline=False)
    help_embed.add_field(name="/ticketpanel", value="🎟️ Post the ticket panel for MM requests.", inline=False)
    help_embed.add_field(name="/tokens @user", value="💰 Check V Tokens and message progress.", inline=False)
    help_embed.add_field(name="/daily", value="🎁 Claim your daily tokens.", inline=False)
    help_embed.add_field(name="/tokenleaderboard", value="📊 See top token holders.", inline=False)
    help_embed.set_footer(text="Use these commands wisely and responsibly!")
    await interaction.response.send_message(embed=help_embed, ephemeral=False)

# ================== ECONOMY / TOKENS ==================
class ShopView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)

    async def buy(self, interaction, cost, item_name):
        user_id = interaction.user.id
        cursor.execute("SELECT tokens, messages, last_daily FROM economy WHERE user_id=?", (user_id,))
        data = cursor.fetchone()
        if data:
            tokens, messages, last_daily = data
        else:
            tokens, messages, last_daily = 0, 0, 0
        if tokens < cost:
            return await interaction.response.send_message("❌ Not enough tokens!", ephemeral=True)
        tokens -= cost
        cursor.execute("INSERT OR REPLACE INTO economy VALUES (?, ?, ?, ?)", (user_id, tokens, messages, last_daily))
        db.commit()
        await interaction.response.send_message(f"✅ Purchased **{item_name}**!", ephemeral=True)

    @discord.ui.button(label="📢 Promotion (7,500)", style=discord.ButtonStyle.blurple)
    async def promo(self, interaction: discord.Interaction, button: discord.ui.Button):
        await self.buy(interaction, 7500, "Server Promotion")

    @discord.ui.button(label="🛡️ Free MM (10,000)", style=discord.ButtonStyle.green)
    async def mm(self, interaction: discord.Interaction, button: discord.ui.Button):
        await self.buy(interaction, 10000, "Free MM Forever")

    @discord.ui.button(label="🎨 Custom Role (20,000)", style=discord.ButtonStyle.gray)
    async def role(self, interaction: discord.Interaction, button: discord.ui.Button):
        await self.buy(interaction, 20000, "Custom Role")

    @discord.ui.button(label="🏪 Open Shop (35,000)", style=discord.ButtonStyle.red)
    async def shop(self, interaction: discord.Interaction, button: discord.ui.Button):
        await self.buy(interaction, 35000, "Open Shop")

    @discord.ui.button(label="🎟️ Priority Ticket (5,000)", style=discord.ButtonStyle.green)
    async def priority_ticket(self, interaction: discord.Interaction, button: discord.ui.Button):
        await self.buy(interaction, 5000, "Priority Ticket")

# ---------------- MESSAGE EVENT / TOKENS -----------------
@bot.event
async def on_message(message):
    if message.author.bot:
        return
    user_id = message.author.id
    now = time.time()
    if user_id in cooldowns and now - cooldowns[user_id] < 1:
        await bot.process_commands(message)
        return
    cooldowns[user_id] = now

    cursor.execute("SELECT tokens, messages, last_daily FROM economy WHERE user_id=?", (user_id,))
    data = cursor.fetchone()
    if data:
        tokens, messages, last_daily = data
    else:
        tokens, messages, last_daily = 0, 0, 0

    messages += 1
    bonus = messages // 2000
    earned = 5 + bonus
    tokens += earned

    cursor.execute("INSERT OR REPLACE INTO economy VALUES (?, ?, ?, ?)", (user_id, tokens, messages, last_daily))
    db.commit()

    # Role progression
    roles_map = [
        (250, "Noob Member"),
        (750, "Semi-Active Member"),
        (2500, "Pro Member"),
        (5500, "Elite Member"),
        (10000, "Godly Member")
    ]
    for req, role_name in roles_map:
        role = discord.utils.get(message.guild.roles, name=role_name)
        if role and messages >= req and role not in message.author.roles:
            await message.author.add_roles(role)

    await bot.process_commands(message)

# ---------------- DAILY REWARD ----------------
@bot.tree.command(name="daily", description="Claim daily tokens")
async def daily(interaction: discord.Interaction):
    user_id = interaction.user.id
    now = int(time.time())
    cursor.execute("SELECT tokens, messages, last_daily FROM economy WHERE user_id=?", (user_id,))
    data = cursor.fetchone()
    if data:
        tokens, messages, last_daily = data
    else:
        tokens, messages, last_daily = 0, 0, 0
    if now - last_daily < 86400:
        return await interaction.response.send_message("❌ You already claimed daily!", ephemeral=True)
    tokens += 500
    cursor.execute("INSERT OR REPLACE INTO economy VALUES (?, ?, ?, ?)", (user_id, tokens, messages, now))
    db.commit()
    await interaction.response.send_message(f"🎁 You claimed **500 V Tokens!**")

# ---------------- TOKEN COMMAND ----------------
@bot.tree.command(name="tokens", description="Check tokens")
@app_commands.describe(user="User to check")
async def tokens(interaction: discord.Interaction, user: discord.Member = None):
    user = user or interaction.user
    cursor.execute("SELECT tokens, messages, last_daily FROM economy WHERE user_id=?", (user.id,))
    data = cursor.fetchone()
    if not data:
        tokens, messages, last_daily = 0, 0, 0
    else:
        tokens, messages, last_daily = data
    bonus = messages // 2000
    next_goal = ((messages // 2000) + 1) * 2000
    bar = progress_bar(messages % 2000, 2000)
    e = embed("💰 V Tokens",
              f"{user.mention}\n\n💸 Tokens: **{tokens}**\n💬 Messages: **{messages}**\n⚡ Per Message: **{5 + bonus}**\n\n📈 Next Boost: {next_goal}\n{bar}")
    await interaction.response.send_message(embed=e, view=ShopView())

# ---------------- TOKEN LEADERBOARD ----------------
@bot.tree.command(name="tokenleaderboard", description="Top token holders")
async def tokenleaderboard(interaction: discord.Interaction):
    cursor.execute("SELECT user_id, tokens FROM economy ORDER BY tokens DESC LIMIT 10")
    data = cursor.fetchall()
    text = ""
    for i, (uid, tokens) in enumerate(data, 1):
        user = await bot.fetch_user(uid)
        text += f"{i}. {user.name} — {tokens}\n"
    await interaction.response.send_message(embed=embed("🏆 Token Leaderboard", text))

# ---------------- READY ----------------
@bot.event
async def on_ready():
    await bot.tree.sync()
    print(f"Logged in as {bot.user}")

bot.run(TOKEN)
