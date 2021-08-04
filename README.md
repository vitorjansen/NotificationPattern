# Notification Pattern

**Objetivo**: Retornar uma coleção de erros.

**Motivações**:

- Evitar estourar exceções (exceções tem maior custo de processamento).

    *Parafraseando Martin Fowler: Exceções são para situações fora do esperado, se você está realizando uma verificação significa que já espera um erro. Então não trate isso como uma exceção.*

- Trafegar erros entre camadas da aplicação (retornar somente true/false nem sempre é interessante pois em caso de mais de um erro o método que recebe o retorno não terá a visibilidade de tudo o que deu errado).

**Funcionamento geral**: Ao fazer uma validação, em caso de erro, este erro é adicionado a uma lista de erros (através de uma classe própria de notificação). E, no momento necessário, esta classe deve retornar a lista acumulada dos erros.

**Abordagem mais simples**

1. Criação da classe que irá gerenciar a lista de erros.

    Entenda esta classe como um gerenciador, ela apenas acumulará os erros e depois os retornará.

    Então esta classe terá uma propriedade que guardará estes erros (uma lista) e dois métodos um para adionar mensagens nesta lista e outro para retornar esta lista quando solicitado.

    ```csharp
    public class Notificador
    {
        // Propriedade que armazena as notificações.
        private List<string> _notificacoes;

        public Notificador()
        {
            _notificacoes = new List<string>();
        }

        // Método que adicionará as mensagens na lista.
        public void AddNotificacao(string mensagem)
        {
            _notificacoes.Add(mensagem);
        }

        // Método que retornará todas as mensagens.
        public List<string> RetornarNotificacoes()
        {
            return _notificacoes;
        }
    }
    ```

4. Injeção de dependência

    Podemos utilizar a nova classe através da injeção de depência. Em caso, de uma API é necessário adiconar ao escopo na classe `Startup`.

    ```csharp
        // Classe que irá utilizar o notificador
        public UsuarioService(Notificador notificador)
        {
            _notificador = notificador;
        }

        // Startup.cs
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddScoped<Notificador>();
        }
    ```

3. Adicionando erros
    
    Ao validar uma instância, adicionar o erro para a lista através do método da classe. (O `return` é chamado apenas na última validação)

    ```csharp
        // Ao invés de retornar apenas false...
        public bool Cadastrar(Usuario novoUsuario)
        {
            if (_usuarioRepository.ValidarExistencia(usuario => usuario.Login.Equals(novoUsuario.Login)))
            {
                return false;
            }

            ***
        }

        // ... Adicionar o erro a lista antes!
        public bool Cadastrar(Usuario novoUsuario)
        {
            if (_usuarioRepository.ValidarExistencia(usuario => usuario.Login.Equals(novoUsuario.Login)))
            {
                AddNotificacao("O e-mail informado já está cadastrado.");
                return false;
            }

            ***
        }
    ```

4. Retornando os erros (Controller em caso de API ou View em caso de aplicação web)

    ```csharp
    if (!_usuarioService.Cadastrar(usuario))
    {
        var erros = _notificador.RetornarNotificacoes();

        return BadRequest(new
        {
            success = false,
            errors = erros
        });
    }
    ```

**Incrementando a abordagem**

1. Abstração da lista

    A lista até aqui é uma coleção apenas de `strings`, porém poderia ser necessário armazenar outro tipo de dado. Vamos abstrair a lista para uma classe específica de `Notificacao`, que aqui continuará sendo apenas uma string, porém podendo ser complementada conforme a necessidade.

    ```csharp
    // Criação da classe
    public class Notificacao
    {
        public string Mensagem { get; set; }

        public Notificacao(string mensagem)
        {
            Mensagem = mensagem;
        }
    }

    // Alterando de string para Notificacao na classe Notificador
    public class Notificador
    {
        private List<Notificacao> _notificacoes;

        public Notificador()
        {
            _notificacoes = new List<Notificacao>();
        }

        public void AddNotificacao(Notificacao notificacao)
        {
            _notificacoes.Add(notificacao);
        }

        public List<Notificacao> RetornarNotificacoes()
        {
            return _notificacoes;
        }
    }

    // Alterando a chamada do método para lançar uma Notificacao ao invés de uma string
    public bool Cadastrar(Usuario novoUsuario)
    {
        if (_usuarioRepository.ValidarExistencia(usuario => usuario.Login.Equals(novoUsuario.Login)))
        {
            _notificador.AddNotificacao(new Notificacao("O e-mail informado já está cadastrado."));
            return false;
        }
    }
    ```

2. Validando a lista antes de retornar

    Para não retornar a lista vazia, é possível criar mais um método para validar se ela está vazia ou não.

    ```csharp
    public class Notificador
    {
        private List<Notificacao> _notificacoes;

        ***

        public bool ExistemNotificacoes()
        {
            return _notificacoes.Any();
        }
    }
    ```

***Extra: Retorno Personalizado/Padrão na Controller***

1. Nova Controller abstrata
    ```csharp
    // Controller princiapal com os métodos de padronização

    [ApiController]
    public class MainController : ControllerBase
    {
        private readonly Notificador _notificador;

        protected MainController(Notificador notificador)
        {
            _notificador = notificador;
        }

        // Valida se existem notificações para serem retornadas
        protected bool OperacaoValida()
        {
            return !_notificador.ExistemNotificacoes();
        }
        
        // Encapsula os erros dentro de uma resposta padrão.
        protected ActionResult RespostaPadronizada(object result = null)
        {
            if (OperacaoValida())
            {
                return Ok(new
                {
                    success = true,
                    data = result
                });
            }

            return BadRequest(new
            {
                success = false,
                errors = _notificador.RetornarNotificacoes().Select(n => n.Mensagem)
            });
        }

        // Construtor que recebe uma ModelState
        // e se ela estiver inválida chama o método que irá capturar estes erros e adicionar a lista.
        // em seguida chama o outro construtor que irá 'montar' a reposta com estes erros
        protected ActionResult RespostaPadronizada(ModelStateDictionary modelState)
        {
            if (!modelState.IsValid) NotificarErroModelState(modelState);
            return RespostaPadronizada();
        }

        // Captura os erros da ModelState e adicina a lista de erros.
        protected void NotificarErroModelState(ModelStateDictionary modelState)
        {
            var erros = modelState.Values.SelectMany(e => e.Errors);
            foreach (var erro in erros)
            {
                var mensagem = erro.Exception == null ? erro.ErrorMessage : erro.Exception.Message;
                NotificarErro(mensagem);
            }
        }

        // Captura os erros da controller ou do método acima e adiciona a lista de erros.
        protected void NotificarErro(string mensagem)
        {
            _notificador.AddNotificacao(new Notificacao(mensagem));
        }
    }
    ```
2. Usando a controller abstrata e o método de padronização

    ```csharp
    // Herda da controller principal
    public class UsuariosController : MainController
    {
        
        // Repassa o notificador para a controller principal
        public UsuariosController(Notificador notificador) : base(notificador)

        ***

        // Utilização no método
        public IActionResult CadastrarUsuario([FromBody] RequestUsuarioDto novoUsuarioDto)
        {
            if (!ModelState.IsValid) return RespostaPadronizada(ModelState);

            var usuario = _mapper.Map<Usuario>(novoUsuarioDto);

            if (!_usuarioService.Cadastrar(usuario)) return RespostaPadronizada(usuario);

            return StatusCode(201);
        }
    ```

***Referências***

*Notification Pattern:*

*https://martinfowler.com/eaaDev/Notification.html*

*Susbstituindo o Throw por Notificatios:*

*https://martinfowler.com/articles/replaceThrowWithNotification.html*
