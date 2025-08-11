const { Sticker, StickerTypes } = require('wa-sticker-formatter');
const baileys = require("@whiskeysockets/baileys");
const makeWASocket = baileys.default;
const { useMultiFileAuthState, downloadContentFromMessage, DisconnectReason, Browsers } = baileys;
const fs = require("fs");
const crypto = require("crypto");
const ytdl = require('ytdl-core');
const ffmpegInstaller = require('node-ffmpeg-installer');
const { exec } = require('child_process');
const path = require('path');
const qrcode = require('qrcode-terminal');
const YoutubeSearchApi = require('youtube-search-api');
const axios = require('axios');

// Configurar ffmpeg
process.env.FFMPEG_PATH = ffmpegInstaller.path;

// Caminhos dos arquivos JSON
const fichasPath = "./fichas.json";
const personagensPath = "./personagens.json";
const gruposPath = "./grupos.json";
const avataresPath = "./avatares.json";

// Funções para ler e salvar JSON
function lerArquivoJSON(caminho) {
  if (!fs.existsSync(caminho)) return {};
  return JSON.parse(fs.readFileSync(caminho, "utf8"));
}

function salvarArquivoJSON(caminho, dados) {
  fs.writeFileSync(caminho, JSON.stringify(dados, null, 2));
}

// Função para embaralhar arrays
function embaralhar(array) {
  for (let i = array.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [array[i], array[j]] = [array[j], array[i]];
  }
  return array;
}

// Função para baixar mídia do WhatsApp (imagem para figurinha)
async function baixarMidia(msg, type = "image") {
  const stream = await downloadContentFromMessage(msg.message[type + "Message"], type);
  let buffer = Buffer.from([]);
  for await (const chunk of stream) {
    buffer = Buffer.concat([buffer, chunk]);
  }
  return buffer;
}

// Função para verificar se o usuário é administrador do grupo
async function verificarAdmin(sock, groupJid, userJid) {
  try {
    const groupMetadata = await sock.groupMetadata(groupJid);
    const isAdmin = groupMetadata.participants.find(p => p.id === userJid && (p.admin === 'admin' || p.admin === 'superadmin'));
    return !!isAdmin;
  } catch (error) {
    console.error("Erro ao verificar admin:", error);
    return false;
  }
}

// Função para pesquisar no YouTube
async function pesquisarYoutube(query) {
  try {
    const result = await YoutubeSearchApi.GetListByKeyword(query, false, 1);
    if (result && result.items && result.items.length > 0) {
      const video = result.items[0];
      return {
        title: video.title,
        url: `https://www.youtube.com/watch?v=${video.id}`,
        thumbnail: video.thumbnail.thumbnails[video.thumbnail.thumbnails.length - 1].url,
        duration: video.length?.simpleText || 'N/A',
        channel: video.channelTitle
      };
    }
    return null;
  } catch (error) {
    console.error('Erro ao pesquisar:', error);
    return null;
  }
}

// Função para baixar thumbnail
async function baixarThumbnail(thumbnailUrl, filename) {
  try {
    const outputPath = path.join(__dirname, 'downloads', `${filename}_thumb.jpg`);
    const response = await axios({
      method: 'GET',
      url: thumbnailUrl,
      responseType: 'stream'
    });

    const writer = fs.createWriteStream(outputPath);
    response.data.pipe(writer);

    return new Promise((resolve, reject) => {
      writer.on('finish', () => resolve(outputPath));
      writer.on('error', reject);
    });
  } catch (error) {
    console.error('Erro ao baixar thumbnail:', error);
    return null;
  }
}

// Função para baixar música do YouTube
async function baixarMusicaYoutube(url, filename) {
  return new Promise((resolve, reject) => {
    const outputPath = path.join(__dirname, 'downloads', `${filename}.mp3`);

    // Criar diretório de downloads se não existir
    if (!fs.existsSync(path.join(__dirname, 'downloads'))) {
      fs.mkdirSync(path.join(__dirname, 'downloads'));
    }

    const ffmpegCommand = `"${ffmpegInstaller.path}" -i pipe:0 -f mp3 -ab 128k -vn "${outputPath}"`;

    const ffmpegProcess = exec(ffmpegCommand, (error) => {
      if (error) {
        console.error('Erro no FFmpeg:', error);
        reject(error);
        return;
      }
      resolve(outputPath);
    });

    const stream = ytdl(url, { 
      filter: 'audioonly',
      quality: 'highestaudio' 
    });

    stream.pipe(ffmpegProcess.stdin);

    stream.on('error', (error) => {
      console.error('Erro no ytdl:', error);
      reject(error);
    });
  });
}

async function startBot() {
  const { state, saveCreds } = await useMultiFileAuthState("./auth_info");

  const sock = makeWASocket({
    auth: state,
    printQRInTerminal: false,
    browser: Browsers.macOS("NejireBot")
  });

  sock.ev.on("creds.update", saveCreds);

  sock.ev.on("connection.update", (update) => {
    const { connection, lastDisconnect, qr } = update;
    if (qr) {
      console.log("📱 Escaneie o QR Code para conectar:");
      console.log("🔗 Código QR:");
      qrcode.generate(qr, { small: true });
      console.log("\n💡 Use o WhatsApp no seu celular para escanear o código acima!");
    }
    if (connection === "close") {
      const reason = lastDisconnect.error?.output?.statusCode;
      if (reason === DisconnectReason.loggedOut) {
        console.log("⚠️ Sessão desconectada. Apague a pasta auth_info e reescaneie o QR.");
      } else {
        console.log("🔄 Conexão perdida, tentando reconectar...");
        startBot();
      }
    }
    if (connection === "open") {
      console.log("✨ Hihi~! A Nejire chegou pra se divertir com vocês! 💙✨");
    }
  });

  sock.ev.on("messages.upsert", async ({ messages, type }) => {
    if (type !== "notify") return;
    const msg = messages[0];
    if (!msg.message || msg.key.fromMe) return;

    const sender = msg.key.remoteJid;
    const isGroup = sender.endsWith("@g.us");

    // Extrair texto da mensagem
    const messageContent = msg.message.conversation
      || msg.message.extendedTextMessage?.text
      || msg.message.imageMessage?.caption
      || "";

    const text = messageContent.trim();

    // Carregar dados de fichas e personagens
    const fichas = lerArquivoJSON(fichasPath);
    const personagens = lerArquivoJSON(personagensPath);

    // Dados do grupo
    let membrosGrupo = [];
    if (isGroup) {
      try {
        const grupoMetadata = await sock.groupMetadata(sender);
        membrosGrupo = grupoMetadata.participants.map(p => p.id);
      } catch {
        membrosGrupo = [];
      }
    }

    // -------------------------------------
    // Comando: !s (Criar figurinha a partir da imagem enviada)
    if (text === "!s") {
      try {
        let bufferSticker;

        if (msg.message.imageMessage) {
          bufferSticker = await baixarMidia(msg, "image");
        } else if (msg.message.extendedTextMessage?.contextInfo?.quotedMessage?.imageMessage) {
          const quoted = msg.message.extendedTextMessage.contextInfo.quotedMessage;
          bufferSticker = await baixarMidia({ message: quoted }, "image");
        } else {
          await sock.sendMessage(sender, { text: "😅 Nee~, manda uma imagem junto com o comando ou responda uma imagem, tá? 💙" });
          return;
        }

        const sticker = new Sticker(bufferSticker, {
          type: StickerTypes.FULL,
          pack: 'Feito com carinho 💙',
          author: 'Bot Nejire',
        });

        const stickerBuffer = await sticker.toBuffer();

        await sock.sendMessage(sender, {
          sticker: stickerBuffer,
        });

        await sock.sendMessage(sender, { text: "✨ Ebaa~! Sua figurinha está prontinha e pode ser baixada! 💖" });
      } catch (e) {
        console.error("Erro ao criar figurinha:", e);
        await sock.sendMessage(sender, { text: "😵‍💫 Ai desculpinha! Não consegui criar a figurinha... tenta de novo, tá bem? 💦" });
      }
      return;
    }

    // -------------------------------------
    // Comando: !dado (roda um dado de 1 a 6)
    if (text === "!dado") {
      const resultado = crypto.randomInt(1, 7);
      const mensagensDado = [
        "🎲 Você rolou *1*! Que azarzinho, mas não desanima, tá? 🌸",
        "🎲 Você rolou *2*! Vamos melhorar, eu acredito em você! 💖",
        "🎲 Você rolou *3*! Um número fofinho, nada mal! 😊",
        "🎲 Você rolou *4*! Tá indo bem, continue assim! ✨",
        "🎲 Você rolou *5*! Uau! Quase perfeito, que linda sorte! 🌟",
        "🎲 Você rolou *6*! Perfeito! Você arrasou demais! 🎉💙"
      ];
      await sock.sendMessage(sender, { text: mensagensDado[resultado - 1] });
      return;
    }

    // -------------------------------------
    // Comando: !comandos (lista de comandos)
    if (text === "!comandos") {
      const listaComandos = 
`🌸 *Comandos da Nejire-chan!* 🌸

🎲 !dado – Role um dado mágico!
📝 !adicionar ficha Nome | Descrição
📄 !ficha Nome
📒 !listar
❌ !remover ficha Nome

🌟 !adicionar personagem Nome - Obra
📚 !lista de personagens
🚫 !remover personagem Número

👥 !adicionar lista de grupos NomeDoGrupo
📋 !lista de grupos

🎭 !adicionar avatar NomeDoAvatar
👤 !lista de avatares
❌ !remover avatar NomeDoAvatar

🖼️ Envie uma imagem com legenda !s – Cria figurinha! 💖

🤣 !rank gay / pau / feio / gostosos / gostosas / carente / aleatório @pessoas
💘 !beijar @pessoa
💀 !matar @pessoa
🦶 !chutar @pessoa

👥 !marcar todos – Marcar todos os membros (apenas admins)
🎵 !play [nome da música e artista] – Pesquisar e baixar música

✨ A Nejire está aqui para alegrar seu dia! 💙`;
      await sock.sendMessage(sender, { text: listaComandos });
      return;
    }

    // -------------------------------------
    // Comando: !marcar todos (apenas para administradores)
    if (text === "!marcar todos") {
      if (!isGroup) {
        await sock.sendMessage(sender, { text: "❌ Este comando só pode ser usado em grupos!" });
        return;
      }

      const autorJid = msg.key.participant || msg.key.remoteJid;
      const isAdmin = await verificarAdmin(sock, sender, autorJid);

      if (!isAdmin) {
        await sock.sendMessage(sender, { text: "🚫 Apenas administradores podem usar este comando!" });
        return;
      }

      try {
        const groupMetadata = await sock.groupMetadata(sender);
        const todosParticipantes = groupMetadata.participants
          .map(p => p.id)
          .filter(jid => jid !== sock.user.id); // Remove o bot da lista

        if (todosParticipantes.length === 0) {
          await sock.sendMessage(sender, { text: "⚠️ Nenhum membro encontrado no grupo!" });
          return;
        }

        const todosMarcados = todosParticipantes.map(jid => "@" + jid.split("@")[0]).join(" ");

        await sock.sendMessage(sender, {
          text: `📢 *ATENÇÃO GALERA!* 📢\n\n${todosMarcados}\n\n👆 Todos marcados pelo admin!`,
          mentions: todosParticipantes
        });
      } catch (error) {
        console.error("Erro ao marcar todos:", error);
        await sock.sendMessage(sender, { text: "❌ Ocorreu um erro ao tentar marcar todos os membros!" });
      }
      return;
    }

    // -------------------------------------
    // Comando: !play (pesquisar e baixar música do YouTube)
    if (text.startsWith("!play ")) {
      const query = text.slice(6).trim();

      if (!query) {
        await sock.sendMessage(sender, { text: "🎵 Digite o nome da música e artista!\nExemplo: !play Bohemian Rhapsody Queen" });
        return;
      }

      try {
        await sock.sendMessage(sender, { text: `🔍 Procurando por: *${query}*\nAguarde um pouquinho... ⏳` });

        // Pesquisar no YouTube
        const videoInfo = await pesquisarYoutube(query);

        if (!videoInfo) {
          await sock.sendMessage(sender, { text: "❌ Não encontrei nenhuma música com esse nome. Tenta com outras palavras?" });
          return;
        }

        await sock.sendMessage(sender, { 
          text: `🎵 Encontrei: *${videoInfo.title}*\n📺 Canal: ${videoInfo.channel}\n⏰ Duração: ${videoInfo.duration}\n\n🎧 Baixando música... Aguarde!` 
        });

        // Criar nome do arquivo seguro
        const safeTitle = videoInfo.title.replace(/[^\w\s]/gi, '').substring(0, 50);
        const filename = `${safeTitle}_${Date.now()}`;

        // Baixar thumbnail e música simultaneamente
        const [thumbnailPath, audioPath] = await Promise.all([
          baixarThumbnail(videoInfo.thumbnail, filename),
          baixarMusicaYoutube(videoInfo.url, filename)
        ]);

        // Preparar dados para envio
        const audioBuffer = fs.readFileSync(audioPath);
        let messageData = {
          audio: audioBuffer,
          mimetype: 'audio/mp4',
          fileName: `${safeTitle}.mp3`,
          caption: `🎵 *${videoInfo.title}*\n📺 ${videoInfo.channel}\n⏰ ${videoInfo.duration}\n\n✨ Baixado com sucesso! 💙`
        };

        // Se conseguiu baixar a thumbnail, adiciona como capa
        if (thumbnailPath && fs.existsSync(thumbnailPath)) {
          const thumbnailBuffer = fs.readFileSync(thumbnailPath);
          messageData.contextInfo = {
            externalAdReply: {
              title: videoInfo.title,
              body: `${videoInfo.channel} • ${videoInfo.duration}`,
              thumbnailUrl: videoInfo.thumbnail,
              sourceUrl: videoInfo.url,
              mediaType: 1,
              renderLargerThumbnail: false
            }
          };
          // Limpar thumbnail
          fs.unlinkSync(thumbnailPath);
        }

        // Enviar o áudio
        await sock.sendMessage(sender, messageData);

        // Limpar arquivo temporário
        fs.unlinkSync(audioPath);

      } catch (error) {
        console.error("Erro ao baixar música:", error);
        await sock.sendMessage(sender, { text: "❌ Não consegui baixar essa música... Tenta outra pesquisa, tá? 💦" });
      }
      return;
    }

    // -------------------------------------
    // Comandos de interação com marcação: !beijar, !matar, !chutar
    if (/^!(beijar|matar|chutar)/.test(text)) {
      const comando = text.split(" ")[0].substring(1);
      const mentions = msg.message.extendedTextMessage?.contextInfo?.mentionedJid || [];
      if (mentions.length === 0) {
        await sock.sendMessage(sender, { text: `🥺 Nee~ você precisa marcar alguém pra usar o comando *!${comando}* 💫` });
        return;
      }
      const alvo = mentions[0];
      const autor = msg.key.participant || msg.key.remoteJid;

      let acao = "";
      let emoji = "";

      if (comando === "beijar") {
        acao = "deu um beijo apaixonado em";
        emoji = "😘💖";
      } else if (comando === "matar") {
        acao = "matou sem piedade";
        emoji = "🔪😵";
      } else if (comando === "chutar") {
        acao = "deu um chutão poderoso em";
        emoji = "🦵🔥";
      }

      await sock.sendMessage(sender, {
        text: `@${autor.split("@")[0]} ${acao} @${alvo.split("@")[0]} ${emoji} 💙`,
        mentions: [autor, alvo]
      });
      return;
    }

    // -------------------------------------
    // RANKS: gay, pau, feio, gostosos, gostosas, carente, aleatório

    const frasesRank = { 
      gay: [
        "{user} é uma bixona nível hard! 🏳️‍🌈🌈",
        "{user} tá mais perdido que cebola em salada de fruta! 😂🍉",
        "{user} planta bananeira com o bumbum na lua! 🍌🍑",
        "{user} se apaixonou pelo Wi-Fi do vizinho! 📶❤️",
        "{user} tá mais colorido que arco-íris duplo! 🌈🌈"
      ],
      pau: [
        "{user} tem um mini pau que nem caberia numa agulha! 😂📏",
        "{user} é discreto, tamanho S na certa! 🤫🍆",
        "{user} disse que é grandão, mas só se for no sonho! 🌙😅",
        "{user} é tamanho médio, nem pequeno, nem grande! ⚖️😎",
        "{user} tá todo orgulhoso do seu troféu! 🏆🍆"
      ],
      feio: [
        "{user} tá feio que só a mamãe gosta! 🤡💔",
        "{user} parece monstro de filme de terror! 👹😱",
        "{user} tem rosto que espanta cachorro! 🐶💨",
        "{user} é feio sim, mas com muito amor! 💖🥺",
        "{user} tem feiúra que virou arte moderna! 🎨👺"
      ],
      gostosos: [
        "{user} é gostoso que nem brigadeiro! 🍫😍",
        "{user} tem corpo de capa de revista! 🏋️‍♂️🔥",
        "{user} deixa todo mundo babando! 🤤💦",
        "{user} é tão gostoso que poderia derreter sorvete! 🍦🥵",
        "{user} é a definição de gostosura! 💯💙"
      ],
      gostosas: [
        "{user} é gostosa que nem cupcake! 🧁😍",
        "{user} arrasa em qualquer lugar! 💃🔥",
        "{user} é como um raio de sol na praia! ☀️🏖️",
        "{user} deixa todo mundo encantado! 😍💫",
        "{user} é a rainha da gostosura! 👑💖"
      ],
      carente: [
        "{user} tá carente e pedindo um abraço! 🤗💙",
        "{user} quer atenção e cafuné! 🥺💞",
        "{user} está carente demais, vamos mimar! 🐾✨",
        "{user} precisa de carinho e chocolate! 🍫❤️",
        "{user} quer ficar no colo até dormir! 😴💕"
      ],
      aleatório: [
        "{user} é uma surpresa toda vez que aparece! 🎉✨",
        "{user} tem um jeitinho único e especial! 🌟😊",
        "{user} é um mistério que encanta a todos! 🕵️‍♂️💖",
        "{user} espalha alegria por onde passa! 🌈😄",
        "{user} é pura magia e fofura! ✨🐰"
      ]
    };

    if (/^!rank (gay|pau|feio|gostosos|gostosas|carente|aleatório|geral)/i.test(text)) {
      const tipoRank = text.split(" ")[1]?.toLowerCase();
      const frases = frasesRank[tipoRank] || frasesRank["aleatório"];
      const isGroup = msg.key.remoteJid.endsWith("@g.us");

      let mentions = msg.message?.extendedTextMessage?.contextInfo?.mentionedJid || [];

      if (tipoRank === "geral") {
        if (!isGroup) {
          await sock.sendMessage(sender, { text: "❌ Esse comando só pode ser usado em grupos!" });
          return;
        }

        try {
          const groupMetadata = await sock.groupMetadata(msg.key.remoteJid);
          const isAdmin = groupMetadata.participants.find(p => p.id === sender && p.admin);

          if (!isAdmin) {
            await sock.sendMessage(sender, { text: "🚫 Apenas administradores podem usar o rank geral!" });
            return;
          }

          mentions = groupMetadata.participants
            .map(p => p.id)
            .filter(jid => jid !== sock.user.id); // remove o bot

          const todosMarcados = mentions.map(jid => "@" + jid.split("@")[0]).join(", ");

          await sock.sendMessage(sender, {
            text: `📢 RANK GERAL DO GRUPO!!\n${todosMarcados}`,
            mentions
          });
          return;
        } catch (e) {
          console.error("Erro no rank geral:", e);
          await sock.sendMessage(sender, { text: "❌ Ocorreu um erro ao tentar pegar os participantes!" });
          return;
        }
      }

      // Limita a marcação manual a no máximo 5 pessoas
      if (mentions.length > 5) {
        await sock.sendMessage(sender, {
          text: "⚠️ Você só pode marcar até 5 pessoas por vez!"
        });
        return;
      }

      // Se não marcar ninguém e for grupo, sorteia até 5 membros aleatórios
      if (mentions.length === 0 && isGroup) {
        try {
          const groupMetadata = await sock.groupMetadata(msg.key.remoteJid);
          const participantes = groupMetadata.participants
            .map(p => p.id)
            .filter(jid => jid !== sock.user.id); // não inclui o bot

          if (participantes.length === 0) {
            await sock.sendMessage(sender, { text: "⚠️ Nenhum membro encontrado no grupo!" });
            return;
          }

          mentions = participantes.sort(() => 0.5 - Math.random()).slice(0, 5); // máximo 5
        } catch (e) {
          console.error("Erro ao pegar membros aleatórios:", e);
          await sock.sendMessage(sender, { text: "❌ Não consegui pegar os membros do grupo." });
          return;
        }
      }

      if (mentions.length === 0) {
        await sock.sendMessage(sender, {
          text: "🥺 Nee~ marca alguém ou usa no grupo pra eu escolher por você!"
        });
        return;
      }

      const mensagens = mentions.map(jid => {
        const userTag = "@" + jid.split("@")[0];
        const frase = frases[Math.floor(Math.random() * frases.length)];
        return frase.replace("{user}", userTag);
      });

      await sock.sendMessage(sender, {
        text: mensagens.join("\n"),
        mentions
      });
      return;
    }


    // -------------------------------------
    // FICHAS: adicionar, listar, mostrar ficha, remover

    if (text.startsWith("!adicionar ficha ")) {
      const dadosFicha = text.slice(16).split("|").map(s => s.trim());
      if (dadosFicha.length < 2) {
        await sock.sendMessage(sender, { text: "💦 Nee~, use o formato correto:\n!adicionar ficha Nome | Descrição" });
        return;
      }
      const nome = dadosFicha[0];
      const descricao = dadosFicha.slice(1).join(" | ");

      fichas[nome.toLowerCase()] = descricao;
      salvarArquivoJSON(fichasPath, fichas);

      await sock.sendMessage(sender, { text: `✨ Ficha de *${nome}* adicionada com sucesso, fofinho! 💖` });
      return;
    }

    if (text === "!listar") {
      const nomesFichas = Object.keys(fichas);
      if (nomesFichas.length === 0) {
        await sock.sendMessage(sender, { text: "📚 Nenhuma ficha cadastrada ainda. Que tal adicionar uma? 🥰" });
        return;
      }
      await sock.sendMessage(sender, { text: "📚 Aqui estão as fichas cadastradas:\n\n" + nomesFichas.map(n => `- ${n}`).join("\n") });
      return;
    }

    if (text.startsWith("!ficha ")) {
      const nome = text.slice(7).toLowerCase();
      if (!fichas[nome]) {
        await sock.sendMessage(sender, { text: `🥺 Não encontrei a ficha *${nome}*. Que tal adicionar com !adicionar ficha? 💙` });
        return;
      }
      await sock.sendMessage(sender, { text: `📄 Ficha *${nome}*:\n\n${fichas[nome]}` });
      return;
    }

    if (text.startsWith("!remover ficha ")) {
      const nome = text.slice(14).toLowerCase();
      if (!fichas[nome]) {
        await sock.sendMessage(sender, { text: `😕 A ficha *${nome}* não existe! Nada pra remover aqui, viu?` });
        return;
      }
      delete fichas[nome];
      salvarArquivoJSON(fichasPath, fichas);
      await sock.sendMessage(sender, { text: `❌ Ficha *${nome}* removida com sucesso. Volte sempre! 💖` });
      return;
    }

    // -------------------------------------
    // PERSONAGENS: adicionar, listar, remover

    if (text.startsWith("!adicionar personagem ")) {
      const dadosPersonagem = text.slice(21).split("-").map(s => s.trim());
      if (dadosPersonagem.length < 2) {
        await sock.sendMessage(sender, { text: "💦 Nee~, use o formato correto:\n!adicionar personagem Nome - Obra" });
        return;
      }
      const nome = dadosPersonagem[0];
      const obra = dadosPersonagem[1];

      personagens[nome.toLowerCase()] = obra;
      salvarArquivoJSON(personagensPath, personagens);

      await sock.sendMessage(sender, { text: `✨ Personagem *${nome}* da obra *${obra}* adicionada com sucesso! 💙` });
      return;
    }

    if (text === "!lista de personagens") {
      const nomesPersonagens = Object.entries(personagens);
      if (nomesPersonagens.length === 0) {
        await sock.sendMessage(sender, { text: "📚 Nenhum personagem cadastrado ainda. Que tal adicionar um? 🥰" });
        return;
      }
      const lista = nomesPersonagens.map(([nome, obra], idx) => `${idx + 1}. ${nome} - ${obra}`).join("\n");
      await sock.sendMessage(sender, { text: "📚 Lista de personagens:\n\n" + lista });
      return;
    }

    if (text.startsWith("!remover personagem ")) {
      const num = parseInt(text.slice(20));
      if (isNaN(num)) {
        await sock.sendMessage(sender, { text: "💦 Número inválido! Use !remover personagem Número" });
        return;
      }
      const nomesPersonagens = Object.keys(personagens);
      if (num < 1 || num > nomesPersonagens.length) {
        await sock.sendMessage(sender, { text: "😕 Número fora da lista. Use !lista de personagens para ver os números." });
        return;
      }
      const nomeRemovido = nomesPersonagens[num - 1];
      delete personagens[nomeRemovido];
      salvarArquivoJSON(personagensPath, personagens);
      await sock.sendMessage(sender, { text: `❌ Personagem *${nomeRemovido}* removido com sucesso. Até logo! 💖` });
      return;
    }

    // -------------------------------------
    // GRUPOS: adicionar, listar

    if (text.startsWith("!adicionar lista de grupos ")) {
      const nomeGrupo = text.slice(27).trim();
      if (!nomeGrupo) {
        await sock.sendMessage(sender, { text: "💦 Use o formato: !adicionar lista de grupos NomeDoGrupo" });
        return;
      }

      const grupos = lerArquivoJSON(gruposPath);
      grupos[nomeGrupo.toLowerCase()] = {
        nome: nomeGrupo,
        adicionadoEm: new Date().toISOString(),
        jid: sender // Salva o JID do grupo atual se for em grupo
      };
      salvarArquivoJSON(gruposPath, grupos);

      await sock.sendMessage(sender, { text: `✨ Grupo *${nomeGrupo}* adicionado à lista com sucesso! 💙` });
      return;
    }

    if (text === "!lista de grupos") {
      const grupos = lerArquivoJSON(gruposPath);
      const nomesGrupos = Object.values(grupos);
      if (nomesGrupos.length === 0) {
        await sock.sendMessage(sender, { text: "📚 Nenhum grupo cadastrado ainda. Que tal adicionar um? 🥰" });
        return;
      }
      const lista = nomesGrupos.map((grupo, idx) => `${idx + 1}. ${grupo.nome}`).join("\n");
      await sock.sendMessage(sender, { text: "📚 Lista de grupos cadastrados:\n\n" + lista });
      return;
    }

    // -------------------------------------
    // AVATARES: adicionar, listar, remover

    if (text.startsWith("!adicionar avatar ")) {
      const nomeAvatar = text.slice(17).trim();
      if (!nomeAvatar) {
        await sock.sendMessage(sender, { text: "💦 Use o formato: !adicionar avatar NomeDoAvatar" });
        return;
      }

      const avatares = lerArquivoJSON(avataresPath);

      // Verificar se avatar já existe na lista
      if (avatares[nomeAvatar.toLowerCase()]) {
        await sock.sendMessage(sender, { text: `❌ O avatar *${nomeAvatar}* já está na lista!` });
        return;
      }

      avatares[nomeAvatar.toLowerCase()] = {
        nome: nomeAvatar,
        adicionadoEm: new Date().toISOString()
      };
      salvarArquivoJSON(avataresPath, avatares);

      await sock.sendMessage(sender, { text: `✨ Avatar *${nomeAvatar}* foi adicionado à lista com sucesso! 💙` });
      return;
    }

    if (text === "!lista de avatares" || text === "!lista de avatares ocupados") {
      const avatares = lerArquivoJSON(avataresPath);
      const avataresList = Object.values(avatares);
      if (avataresList.length === 0) {
        await sock.sendMessage(sender, { text: "📚 Nenhum avatar na lista ainda. Que tal adicionar um? 🥰" });
        return;
      }
      const lista = avataresList.map((avatar, idx) => 
        `${idx + 1}. ${avatar.nome}`
      ).join("\n");
      await sock.sendMessage(sender, { text: "📚 Lista de avatares:\n\n" + lista });
      return;
    }

    if (text.startsWith("!remover avatar ")) {
      const nomeAvatar = text.slice(15).trim();
      if (!nomeAvatar) {
        await sock.sendMessage(sender, { text: "💦 Use o formato: !remover avatar NomeDoAvatar" });
        return;
      }

      const avatares = lerArquivoJSON(avataresPath);

      if (!avatares[nomeAvatar.toLowerCase()]) {
        await sock.sendMessage(sender, { text: `😕 O avatar *${nomeAvatar}* não está na lista!` });
        return;
      }

      delete avatares[nomeAvatar.toLowerCase()];
      salvarArquivoJSON(avataresPath, avatares);
      await sock.sendMessage(sender, { text: `❌ Avatar *${nomeAvatar}* foi removido da lista com sucesso! 💖` });
      return;
    }

    // -------------------------------------
    // Caso comando não reconhecido e comece com "!"
    if (text.startsWith("!")) {
      await sock.sendMessage(sender, { text: "🤔 Ops, não entendi esse comando... Será que digitou errado? Tenta outra vez, tá? 💙" });
      return;
    }

  });

}

startBot(); 
