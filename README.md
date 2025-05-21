using System;
using System.Threading.Tasks;

namespace MensageriaApp
{
    // Domain Models
    public abstract class Mensagem
    {
        public string Conteudo { get; }
        public DateTime DataEnvio { get; } = DateTime.UtcNow;
        
        protected Mensagem(string conteudo)
        {
            Conteudo = conteudo ?? throw new ArgumentNullException(nameof(conteudo));
        }
        
        public abstract string FormatacaoEspecifica();
    }

    public abstract class MensagemMidia : Mensagem
    {
        public string NomeArquivo { get; }
        public string FormatoArquivo { get; }
        
        protected MensagemMidia(string conteudo, string nomeArquivo, string formatoArquivo) 
            : base(conteudo)
        {
            NomeArquivo = nomeArquivo ?? throw new ArgumentNullException(nameof(nomeArquivo));
            FormatoArquivo = formatoArquivo ?? throw new ArgumentNullException(nameof(formatoArquivo));
        }
    }

    public class MensagemTexto : Mensagem
    {
        public MensagemTexto(string conteudo) : base(conteudo) { }
        
        public override string FormatacaoEspecifica() => $"[TEXTO] {Conteudo}";
    }

    public class MensagemVideo : MensagemMidia
    {
        public TimeSpan Duracao { get; }
        
        public MensagemVideo(string conteudo, string nomeArquivo, string formatoArquivo, TimeSpan duracao)
            : base(conteudo, nomeArquivo, formatoArquivo)
        {
            if (duracao <= TimeSpan.Zero)
                throw new ArgumentException("Duração deve ser positiva", nameof(duracao));
                
            Duracao = duracao;
        }
        
        public override string FormatacaoEspecifica() => 
            $"[VÍDEO] {Conteudo} | Arquivo: {NomeArquivo}.{FormatoArquivo} | Duração: {Duracao.TotalSeconds}s";
    }

    // Services
    public interface ICanalComunicacao
    {
        Task EnviarMensagemAsync(Mensagem mensagem);
    }

    public abstract class CanalBase : ICanalComunicacao
    {
        public string Destinatario { get; }
        
        protected CanalBase(string destinatario)
        {
            if (string.IsNullOrWhiteSpace(destinatario))
                throw new ArgumentNullException(nameof(destinatario));
                
            Destinatario = destinatario;
        }
        
        public abstract Task EnviarMensagemAsync(Mensagem mensagem);
        
        protected virtual string FormatacaoMensagem(Mensagem mensagem)
        {
            return $"{DateTime.UtcNow:yyyy-MM-dd HH:mm:ss} | Para: {Destinatario} | {mensagem.FormatacaoEspecifica()}";
        }
    }

    public class WhatsAppService : CanalBase
    {
        public WhatsAppService(string numero) : base(ValidarNumero(numero)) { }
        
        public override async Task EnviarMensagemAsync(Mensagem mensagem)
        {
            if (mensagem == null) throw new ArgumentNullException(nameof(mensagem));
            
            // Simulação de envio assíncrono
            await Task.Delay(100); 
            Console.WriteLine($"WhatsApp: {FormatacaoMensagem(mensagem)}");
        }
        
        private static string ValidarNumero(string numero)
        {
            // Lógica de validação real iria aqui
            if (string.IsNullOrWhiteSpace(numero) || !numero.StartsWith("+"))
                throw new ArgumentException("Número de WhatsApp inválido", nameof(numero));
                
            return numero;
        }
    }

    // Factory
    public static class CanalFactory
    {
        public static ICanalComunicacao CriarCanal(TipoCanal tipo, string destinatario)
        {
            return tipo switch
            {
                TipoCanal.WhatsApp => new WhatsAppService(destinatario),
                // Outros canais...
                _ => throw new NotImplementedException($"Canal {tipo} não implementado")
            };
        }
    }

    public enum TipoCanal { WhatsApp, Telegram, Facebook, Instagram }

    // Logger (Adapter pattern)
    public interface ILogger
    {
        void Log(string message);
    }

    public class ConsoleLogger : ILogger
    {
        public void Log(string message) => Console.WriteLine(message);
    }

    // Usage
    class Program
    {
        static async Task Main()
        {
            try
            {
                var logger = new ConsoleLogger();
                logger.Log("Iniciando envio de mensagens...");

                var canal = CanalFactory.CriarCanal(TipoCanal.WhatsApp, "+5511999999999");
                
                var mensagens = new Mensagem[]
                {
                    new MensagemTexto("Olá, tudo bem?"),
                    new MensagemVideo(
                        "Veja este vídeo", 
                        "video1", 
                        "mp4", 
                        TimeSpan.FromSeconds(120))
                };

                foreach (var msg in mensagens)
                {
                    await canal.EnviarMensagemAsync(msg);
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Erro: {ex.Message}");
            }
        }
    }
}
