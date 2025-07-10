# Guia de Testabilidade para Aplicações .NET

A testabilidade no .NET é crucial para construir sistemas robustos e escaláveis, especialmente em um contexto de arquitetura baseada em APIs e integrações. O foco principal é no isolamento das unidades de código e na injeção de dependência para facilitar a substituição de componentes reais por mocks ou stubs em tempo de teste.

## 1. Princípios de Testabilidade em .NET

### Injeção de Dependência (DI)

Fundamental. A maioria dos frameworks .NET (ASP.NET Core) tem DI integrada. Isso permite que dependências (como repositórios, clientes Kafka, outros serviços) sejam fornecidas externamente, em vez de serem instanciadas diretamente dentro das classes.

### Interfaces

Definir interfaces para suas dependências (ex: `ITransacaoRepository`, `IKafkaProducer`). Isso permite que você crie múltiplas implementações para a mesma funcionalidade (uma real, uma mockada para testes) e as "troque" no momento da configuração da aplicação.

### Single Responsibility Principle (SRP)

Classes e métodos devem ter uma única responsabilidade. Isso simplifica o teste, pois cada teste pode focar em uma funcionalidade específica.

### Testes Automatizados em Camadas

- **Testes de Unidade**: Pequenas unidades de código com todas as dependências externas mockadas. Devem ser muito rápidos.
- **Testes de Integração**: Validam a interação entre componentes (ex: API com DB, API com Kafka), utilizando dependências reais ou contêineres descartáveis.
- **Testes de Componente/API**: Testam a API como um todo, isolando as dependências externas ou usando contêineres.
- **Testes End-to-End (E2E)**: Testam o sistema completo (frontend e backend).

## 2. Ferramentas Essenciais

- **xUnit / NUnit / MSTest**: Frameworks para testes de unidade e integração.
- **Moq / NSubstitute**: Mocking de dependências.
- **Microsoft.AspNetCore.Mvc.Testing**: Cria servidor em memória para testes HTTP.
- **Testcontainers (.NET)**: Provisiona contêineres Docker descartáveis para testes de integração.
- **Postman / Thunder Client**: Testes manuais de API.

## 3. Aplicando Testabilidade: Exemplos Práticos

### 3.1. Injeção de Dependência e Interfaces

```csharp
// ITransacaoRepository.cs
public interface ITransacaoRepository
{
    Task<bool> SalvarTransacao(Transacao transacao);
    Task<Transacao?> ObterTransacao(string id);
}

// IKafkaProducer.cs
public interface IKafkaProducer
{
    Task PublicarMensagem(string topic, string message);
}
```

### 3.2. Implementações Reais

```csharp
public class TransacaoRepository : ITransacaoRepository { /* DB real */ }
public class KafkaProducer : IKafkaProducer { /* Kafka real */ }
```

### 3.3. Implementações Mockadas

```csharp
public class InMemoryTransacaoRepository : ITransacaoRepository { /* memória */ }
public class MockKafkaProducer : IKafkaProducer { /* simulado */ }
```

### 3.4. Registro de Dependências por Ambiente

```csharp
if (useMocksForExternalServices)
{
    services.AddSingleton<ITransacaoRepository, InMemoryTransacaoRepository>();
    services.AddSingleton<IKafkaProducer, MockKafkaProducer>();
}
else
{
    services.AddSingleton<ITransacaoRepository, TransacaoRepository>();
    services.AddSingleton<IKafkaProducer, KafkaProducer>();
}
```

### 3.5. Testes de Unidade com Moq

```csharp
[Fact]
public async Task ProcessarTransacao_ShouldSaveAndPublishMessage_WhenValid()
{
    var mockRepo = new Mock<ITransacaoRepository>();
    var mockKafka = new Mock<IKafkaProducer>();
    var service = new TransacaoService(mockRepo.Object, mockKafka.Object);
    var transacao = new Transacao { Id = "tx123", Valor = 100.00m };

    mockRepo.Setup(r => r.SalvarTransacao(It.IsAny<Transacao>())).ReturnsAsync(true);
    mockKafka.Setup(k => k.PublicarMensagem(It.IsAny<string>(), It.IsAny<string>())).Returns(Task.CompletedTask);

    var result = await service.ProcessarTransacao(transacao);

    Assert.True(result);
    mockRepo.Verify(r => r.SalvarTransacao(transacao), Times.Once);
    mockKafka.Verify(k => k.PublicarMensagem("transacoes-topic", It.IsAny<string>()), Times.Once);
}
```

### 3.6. Testes de Integração com Testcontainers

```csharp
public class TransacaoIntegrationTests : IAsyncLifetime
{
    // Setup containers PostgreSQL e Kafka
    // Criação da API com WebApplicationFactory e configurações injectadas
    // Testa persistência no banco e mensagem enviada ao Kafka
}
```

## 4. Boas Práticas para Testabilidade em .NET

- Priorize testes de unidade: baratos e rápidos.
- Use DI de forma consistente.
- Configure `appsettings` para ambientes de teste.
- Use `WebApplicationFactory` para testes HTTP em memória.
- Use Testcontainers para integração real com DB/Kafka.
- Monitore cobertura com Coverlet.
- Rode testes no CI para feedback rápido.

Aplicar esses conceitos facilita a escrita de testes e força um design limpo e modular, resultando em software mais robusto e de fácil manutenção.
