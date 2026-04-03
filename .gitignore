require('dotenv').config();
const { Client, GatewayIntentBits, EmbedBuilder, PermissionsBitField, ActivityType, SlashCommandBuilder, REST, Routes } = require('discord.js');
const { joinVoiceChannel, createAudioPlayer, createAudioResource, AudioPlayerStatus } = require('@discordjs/voice');
const ytdl = require('@distube/ytdl-core');
const ytSearch = require('yt-search');

// ==================== KHỞI TẠO CLIENT ====================
const client = new Client({
    intents: [
        GatewayIntentBits.Guilds,
        GatewayIntentBits.GuildMessages,
        GatewayIntentBits.MessageContent,
        GatewayIntentBits.GuildVoiceStates,
        GatewayIntentBits.GuildMembers,
    ]
});

// ==================== CẤU HÌNH TỪ ENV ====================
const TOKEN = process.env.TOKEN;
const DEFAULT_AUTO_CHANNEL = process.env.AUTO_CHANNEL_ID;
const DEFAULT_AUTO_MESSAGE = process.env.AUTO_MESSAGE || 'Tin nhắn tự động mặc định';
const DEFAULT_AUTO_INTERVAL = parseInt(process.env.AUTO_INTERVAL) || 60000;

// Trạng thái auto message
let autoEnabled = false;
let autoChannelId = DEFAULT_AUTO_CHANNEL;
let autoMessage = DEFAULT_AUTO_MESSAGE;
let autoIntervalMs = DEFAULT_AUTO_INTERVAL;
let autoIntervalObj = null;

// Music queue: Map<guildId, queue>
const queues = new Map();

// ==================== HÀM AUTO MESSAGE ====================
async function sendAutoMessage() {
    if (!autoEnabled || !autoChannelId) return;
    try {
        const channel = await client.channels.fetch(autoChannelId);
        if (!channel || !channel.isTextBased()) return;
        await channel.send(autoMessage);
        console.log(`[${new Date().toLocaleString()}] Auto message: "${autoMessage}"`);
    } catch (err) {
        console.error('Auto message error:', err);
    }
}

function startAutoMessage() {
    if (autoIntervalObj) clearInterval(autoIntervalObj);
    autoEnabled = true;
    autoIntervalObj = setInterval(sendAutoMessage, autoIntervalMs);
    console.log(`✅ Auto message started (interval: ${autoIntervalMs}ms, channel: ${autoChannelId})`);
}

function stopAutoMessage() {
    if (autoIntervalObj) {
        clearInterval(autoIntervalObj);
        autoIntervalObj = null;
    }
    autoEnabled = false;
    console.log('⏹️ Auto message stopped');
}

// ==================== HÀM MUSIC ====================
function getQueue(guildId) {
    if (!queues.has(guildId)) {
        queues.set(guildId, { songs: [], player: null, connection: null, currentIndex: -1, volume: 100 });
    }
    return queues.get(guildId);
}

async function playSong(guildId, song) {
    const queue = getQueue(guildId);
    if (!song) {
        if (queue.connection) queue.connection.destroy();
        queues.delete(guildId);
        return;
    }
    try {
        const stream = ytdl(song.url, { filter: 'audioonly', quality: 'highestaudio', highWaterMark: 1 << 25 });
        const resource = createAudioResource(stream, { inlineVolume: true });
        resource.volume.setVolumeLogarithmic(queue.volume / 100);
        
        if (!queue.player) {
            queue.player = createAudioPlayer();
            queue.player.on(AudioPlayerStatus.Idle, () => {
                queue.currentIndex++;
                if (queue.currentIndex < queue.songs.length) {
                    playSong(guildId, queue.songs[queue.currentIndex]);
                } else {
                    queue.songs = [];
                    queue.currentIndex = -1;
                    if (queue.connection) queue.connection.destroy();
                    queues.delete(guildId);
                }
            });
            queue.player.on('error', err => console.error('Player error:', err));
        }
        queue.player.play(resource);
        if (queue.connection) queue.connection.subscribe(queue.player);
    } catch (err) {
        console.error('Play error:', err);
    }
}

// ==================== ĐĂNG KÝ SLASH COMMANDS ====================
const commands = [
    // Auto message
    new SlashCommandBuilder().setName('autostart').setDescription('Bắt đầu gửi tin nhắn tự động'),
    new SlashCommandBuilder().setName('autostop').setDescription('Dừng gửi tin nhắn tự động'),
    new SlashCommandBuilder().setName('autoset')
        .setDescription('Cấu hình auto message')
        .addSubcommand(sub => sub.setName('channel').setDescription('Đổi kênh đích').addStringOption(opt => opt.setName('id').setDescription('ID kênh').setRequired(true)))
        .addSubcommand(sub => sub.setName('message').setDescription('Đổi nội dung tin nhắn').addStringOption(opt => opt.setName('content').setDescription('Nội dung').setRequired(true)))
        .addSubcommand(sub => sub.setName('interval').setDescription('Đổi khoảng thời gian (giây)').addIntegerOption(opt => opt.setName('seconds').setDescription('Giây').setRequired(true))),
    
    // Music
    new SlashCommandBuilder().setName('play').setDescription('Phát nhạc từ YouTube (URL hoặc tên bài hát)').addStringOption(opt => opt.setName('query').setDescription('Tên bài hát hoặc URL').setRequired(true)),
    new SlashCommandBuilder().setName('skip').setDescription('Bỏ qua bài hiện tại'),
    new SlashCommandBuilder().setName('stop').setDescription('Dừng phát nhạc và rời khỏi voice'),
    new SlashCommandBuilder().setName('queue').setDescription('Xem danh sách chờ'),
    new SlashCommandBuilder().setName('volume').setDescription('Chỉnh âm lượng (0-200)').addIntegerOption(opt => opt.setName('level').setDescription('Phần trăm').setRequired(true)),
    new SlashCommandBuilder().setName('nowplaying').setDescription('Bài hát đang phát'),
    
    // Moderation
    new SlashCommandBuilder().setName('kick').setDescription('Kick thành viên').addUserOption(opt => opt.setName('target').setDescription('Người dùng').setRequired(true)),
    new SlashCommandBuilder().setName('ban').setDescription('Ban thành viên').addUserOption(opt => opt.setName('target').setDescription('Người dùng').setRequired(true)),
    new SlashCommandBuilder().setName('clear').setDescription('Xóa tin nhắn hàng loạt').addIntegerOption(opt => opt.setName('amount').setDescription('Số lượng (1-100)').setRequired(true)),
    
    // Fun & Utility
    new SlashCommandBuilder().setName('ping').setDescription('Kiểm tra độ trễ của bot'),
    new SlashCommandBuilder().setName('fun').setDescription('Nhận một câu nói hài hước ngẫu nhiên'),
    new SlashCommandBuilder().setName('help').setDescription('Hiển thị hướng dẫn sử dụng'),
];

// ==================== XỬ LÝ INTERACTION ====================
client.on('interactionCreate', async (interaction) => {
    if (!interaction.isChatInputCommand()) return;
    const { commandName, options, guild, member, channel } = interaction;
    const voiceChannel = member.voice.channel;

    // -------------------- AUTO MESSAGE --------------------
    if (commandName === 'autostart') {
        if (autoEnabled) return interaction.reply({ content: 'Auto message đã đang chạy!', ephemeral: true });
        startAutoMessage();
        return interaction.reply(`✅ Đã bắt đầu auto message (kênh <#${autoChannelId}>, nội dung: "${autoMessage}", interval: ${autoIntervalMs/1000}s)`);
    }
    if (commandName === 'autostop') {
        if (!autoEnabled) return interaction.reply({ content: 'Auto message chưa được bật!', ephemeral: true });
        stopAutoMessage();
        return interaction.reply('⏹️ Đã dừng auto message.');
    }
    if (commandName === 'autoset') {
        const sub = options.getSubcommand();
        if (sub === 'channel') {
            const newId = options.getString('id');
            try {
                const testChannel = await client.channels.fetch(newId);
                if (!testChannel || !testChannel.isTextBased()) throw new Error();
                autoChannelId = newId;
                if (autoEnabled) { stopAutoMessage(); startAutoMessage(); }
                await interaction.reply(`📡 Đã đổi kênh auto message thành <#${newId}>`);
            } catch { interaction.reply({ content: 'ID kênh không hợp lệ!', ephemeral: true }); }
        } else if (sub === 'message') {
            autoMessage = options.getString('content');
            await interaction.reply(`✏️ Nội dung auto message mới: "${autoMessage}"`);
        } else if (sub === 'interval') {
            const sec = options.getInteger('seconds');
            if (sec < 1) return interaction.reply({ content: 'Interval phải lớn hơn 0 giây!', ephemeral: true });
            autoIntervalMs = sec * 1000;
            if (autoEnabled) { stopAutoMessage(); startAutoMessage(); }
            await interaction.reply(`⏱️ Đã đặt interval thành ${sec} giây`);
        }
    }

    // -------------------- MUSIC --------------------
    if (commandName === 'play') {
        if (!voiceChannel) return interaction.reply({ content: 'Bạn phải ở trong voice channel!', ephemeral: true });
        const query = options.getString('query');
        await interaction.deferReply();
        let songInfo = null;
        if (query.startsWith('http')) {
            try {
                songInfo = await ytdl.getInfo(query);
            } catch { return interaction.editReply('URL không hợp lệ!'); }
        } else {
            const searchResult = await ytSearch(query);
            if (!searchResult.videos.length) return interaction.editReply('Không tìm thấy bài hát nào!');
            songInfo = await ytdl.getInfo(searchResult.videos[0].url);
        }
        const song = { title: songInfo.videoDetails.title, url: songInfo.videoDetails.video_url };
        const queue = getQueue(guild.id);
        if (queue.songs.length === 0) {
            queue.songs.push(song);
            queue.currentIndex = 0;
            const connection = joinVoiceChannel({ channelId: voiceChannel.id, guildId: guild.id, adapterCreator: guild.voiceAdapterCreator });
            queue.connection = connection;
            await playSong(guild.id, song);
            await interaction.editReply(`🎵 Đang phát: **${song.title}**`);
        } else {
            queue.songs.push(song);
            await interaction.editReply(`✅ Đã thêm **${song.title}** vào hàng đợi (vị trí ${queue.songs.length})`);
        }
    }
    if (commandName === 'skip') {
        const queue = getQueue(guild.id);
        if (!queue.player || queue.player.state.status !== 'playing') return interaction.reply({ content: 'Không có bài nào đang phát!', ephemeral: true });
        queue.player.stop();
        await interaction.reply('⏭️ Đã bỏ qua bài hiện tại');
    }
    if (commandName === 'stop') {
        const queue = getQueue(guild.id);
        if (queue.connection) {
            queue.connection.destroy();
            queues.delete(guild.id);
            await interaction.reply('⏹️ Đã dừng nhạc và rời khỏi voice channel');
        } else {
            await interaction.reply({ content: 'Bot không ở trong voice channel!', ephemeral: true });
        }
    }
    if (commandName === 'queue') {
        const queue = getQueue(guild.id);
        if (queue.songs.length === 0) return interaction.reply({ content: 'Hàng đợi trống.', ephemeral: true });
        let list = '**Hàng đợi hiện tại:**\n';
        queue.songs.forEach((s, i) => { list += `${i+1}. ${s.title}\n`; });
        await interaction.reply(list);
    }
    if (commandName === 'volume') {
        const vol = options.getInteger('level');
        if (vol < 0 || vol > 200) return interaction.reply({ content: 'Volume phải từ 0 đến 200!', ephemeral: true });
        const queue = getQueue(guild.id);
        queue.volume = vol;
        if (queue.player && queue.player.state.resource) queue.player.state.resource.volume.setVolumeLogarithmic(vol/100);
        await interaction.reply(`🔊 Đã chỉnh volume thành ${vol}%`);
    }
    if (commandName === 'nowplaying') {
        const queue = getQueue(guild.id);
        if (queue.currentIndex >= 0 && queue.songs[queue.currentIndex]) {
            await interaction.reply(`🎶 Đang phát: **${queue.songs[queue.currentIndex].title}**`);
        } else {
            await interaction.reply({ content: 'Không có bài nào đang phát.', ephemeral: true });
        }
    }

    // -------------------- MODERATION --------------------
    if (commandName === 'kick') {
        if (!member.permissions.has(PermissionsBitField.Flags.KickMembers)) return interaction.reply({ content: 'Bạn không có quyền kick!', ephemeral: true });
        const target = options.getMember('target');
        if (!target) return interaction.reply({ content: 'Không tìm thấy người dùng!', ephemeral: true });
        if (!target.kickable) return interaction.reply({ content: 'Bot không thể kick người này!', ephemeral: true });
        await target.kick();
        await interaction.reply(`👢 Đã kick ${target.user.tag}`);
    }
    if (commandName === 'ban') {
        if (!member.permissions.has(PermissionsBitField.Flags.BanMembers)) return interaction.reply({ content: 'Bạn không có quyền ban!', ephemeral: true });
        const target = options.getMember('target');
        if (!target) return interaction.reply({ content: 'Không tìm thấy người dùng!', ephemeral: true });
        if (!target.bannable) return interaction.reply({ content: 'Bot không thể ban người này!', ephemeral: true });
        await target.ban();
        await interaction.reply(`🔨 Đã ban ${target.user.tag}`);
    }
    if (commandName === 'clear') {
        if (!member.permissions.has(PermissionsBitField.Flags.ManageMessages)) return interaction.reply({ content: 'Bạn cần quyền ManageMessages!', ephemeral: true });
        const amount = options.getInteger('amount');
        if (amount < 1 || amount > 100) return interaction.reply({ content: 'Số lượng từ 1 đến 100!', ephemeral: true });
        await channel.bulkDelete(amount, true);
        await interaction.reply({ content: `🧹 Đã xóa ${amount} tin nhắn`, ephemeral: true });
    }

    // -------------------- FUN & UTILITY --------------------
    if (commandName === 'ping') {
        await interaction.reply(`🏓 Pong! Độ trễ: ${client.ws.ping}ms`);
    }
    if (commandName === 'fun') {
        const jokes = [
            "Tại sao lập trình viên hay nhầm Halloween và Christmas? Vì Oct 31 = Dec 25!",
            "Máy tính của tôi có vấn đề: nó không hiểu tại sao tôi lại cần nó khi tôi có điện thoại.",
            "Discord bot nói: 'Tôi không phải là người yêu của bạn, nhưng tôi luôn online chờ bạn!'",
            "Hôm qua tôi thử ăn USB, nhưng nó báo lỗi: Không tìm thấy bootable device.",
            "Bạn: /play nhạc buồn. Bot: Chơi nhạc buồn thì tôi cũng buồn lây."
        ];
        const randomJoke = jokes[Math.floor(Math.random() * jokes.length)];
        await interaction.reply(randomJoke);
    }
    if (commandName === 'help') {
        const embed = new EmbedBuilder()
            .setColor(0x00AE86)
            .setTitle('🤖 Bot Đa Chức Năng (Slash Commands)')
            .addFields(
                { name: '⏰ Auto Message', value: '`/autostart` - Bắt đầu\n`/autostop` - Dừng\n`/autoset channel/id/message/interval` - Cấu hình', inline: false },
                { name: '🎵 Music', value: '`/play` - Phát nhạc (tên hoặc URL)\n`/skip` - Bỏ qua\n`/stop` - Dừng\n`/queue` - Xem hàng đợi\n`/volume` - Chỉnh âm lượng\n`/nowplaying` - Bài đang phát', inline: false },
                { name: '🛠️ Moderation', value: '`/kick` - Kick member\n`/ban` - Ban member\n`/clear` - Xóa tin nhắn', inline: false },
                { name: '😄 Giải trí', value: '`/ping` - Kiểm tra ping\n`/fun` - Câu hài hước\n`/help` - Trợ giúp này', inline: false }
            )
            .setFooter({ text: 'Sử dụng các lệnh slash (gõ /)' });
        await interaction.reply({ embeds: [embed] });
    }
});

// ==================== ĐĂNG KÝ COMMANDS LÊN DISCORD ====================
client.once('ready', async () => {
    console.log(`✅ Bot online: ${client.user.tag}`);
    client.user.setPresence({ activities: [{ name: '/help | /play', type: ActivityType.Listening }], status: 'online' });
    const rest = new REST({ version: '10' }).setToken(TOKEN);
    try {
        console.log('🔄 Đang đăng ký slash commands...');
        await rest.put(Routes.applicationCommands(client.user.id), { body: commands.map(cmd => cmd.toJSON()) });
        console.log('✅ Đã đăng ký thành công!');
    } catch (err) { console.error('Lỗi đăng ký commands:', err); }
});

// ==================== ĐĂNG NHẬP ====================
client.login(TOKEN).catch(err => { console.error('Login error:', err); process.exit(1); });

process.on('SIGINT', () => {
    if (autoIntervalObj) clearInterval(autoIntervalObj);
    console.log('Shutting down...');
    process.exit(0);
});
