# Guia de Instrumentação Manual - OpenTelemetry (NestJS)

Este guia documenta como criar Spans customizados e registrar erros manualmente na sua aplicação backend (NestJS) para que eles apareçam perfeitamente correlacionados no Grafana (Tempo + Loki).

## 1. Nomenclatura de Ambientes
Para facilitar a filtragem nos dashboards do Grafana, utilize sempre prefixos curtos na variável de ambiente de definição do serviço (`process.env.ENV`). 

Padrão recomendado:
- `DEV` (Desenvolvimento)
- `HML` (Homologação)
- `PRD` (Produção)

No seu arquivo `tracing.ts`, isso garante que o `serviceName` fique limpo (ex: `sigapro-api-PRD`), organizando nativamente os seus logs via OTLP.

## 2. Como criar Spans Customizados e Registrar Erros

A melhor prática de observabilidade não é apenas jogar um log solto quando ocorre um erro, mas sim **anexar a exceção ao Span atual**. Isso faz com que a interface do Grafana Tempo mostre o erro visualmente atrelado ao exato momento e fluxo da requisição.

### Passo a Passo no Código

1. Importe a API do OpenTelemetry no seu arquivo da API:
```typescript
import { trace, SpanStatusCode } from '@opentelemetry/api';
```

2. Envolva o bloco de código crítico usando `startActiveSpan`, `recordException` e garanta o fechamento com `span.end()`:

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { trace, SpanStatusCode } from '@opentelemetry/api';

@Injectable()
export class SuaService {
  private readonly logger = new Logger(SuaService.name);
  
  // 1. Cria uma instância do "Tracer" nomeada (Geralmente o nome do módulo ou domínio)
  private readonly tracer = trace.getTracer('sigapro-meu-modulo-customizado');

  async executarAcaoComplexa(dados: any) {
    // 2. Cria e inicia um Span ativo
    return await this.tracer.startActiveSpan('nome-do-meu-span-manual', async (span) => {
      try {
        // --- Sua lógica de negócio entra aqui ---
        
        // Adiciona atributos úteis para facilitar buscas e filtros no Grafana
        span.setAttribute('usuario.id', dados.userId);
        span.setAttribute('acao.tipo', 'processamento_pesado');

        const resultado = await regraDeNegocioQuePodeFalhar(dados);

        // Marca o span com sucesso se tudo ocorreu bem
        span.setStatus({ code: SpanStatusCode.OK });
        return resultado;

      } catch (error) {
        // 3. MAGIA DO OTLP: Grava a StackTrace e a mensagem de erro DENTRO do Span!
        span.recordException(error);
        
        // 4. Marca o Status do Span como "ERROR" (Ficará vermelho no Grafana Tempo)
        span.setStatus({
          code: SpanStatusCode.ERROR,
          message: error.message,
        });

        // Opcional: Mantém o log tradicional para visualização rápida no terminal do Docker
        this.logger.error(`Falha ao executar ação: ${error.message}`, error.stack);

        throw error; // Repassa o erro para ser tratado pela Controller ou Exception Filter
      } finally {
        // 5. CRÍTICO: Sempre feche o span no bloco finally. 
        // Spans não fechados causam memory leak e nunca chegam ao Collector.
        span.end();
      }
    });
  }
}
```

## 3. O que esperar no Grafana?
- Na aba **Trace**, a execução desse método aparecerá como uma barra separada chamada `nome-do-meu-span-manual`.
- Se a função falhar, essa barra ficará destacada em **VERMELHO**.
- Ao clicar na barra vermelha, uma seção chamada **"Events"** (ou Logs) revelará a StackTrace completa anexada pela função `recordException()`, permitindo um debug cirúrgico sem precisar buscar logs perdidos pelo ID.
