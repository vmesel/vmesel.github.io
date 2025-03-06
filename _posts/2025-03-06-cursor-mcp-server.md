---
published: true
title: AI and Software Engineering Careers - How I guess everything will change
layout: post
tags: [ai, career, software engineering]
categories: [ai, software engineering]
permalink: ai-and-software-engineering-careers
thumbnail: /assets/imgs/prompt-engineering.jpg
---

# Construindo um Servidor MCP com FastMCP: Guia Prático e Rápido!

Se você já quis integrar um modelo de linguagem com APIs externas de forma eficiente, então este post é para você! Vamos explorar como construir um servidor MCP (Model Context Protocol) usando **FastMCP**, um servidor base que facilita essa implementação. Esse servidor permitirá que você conecte ferramentas externas ao LLM, tornando-o um verdadeiro agente interativo com acesso a APIs externas.

## O que é MCP?

O MCP (Model Context Protocol) é um protocolo que permite a interação entre um modelo de linguagem e ferramentas externas. Ele pode funcionar de duas formas:
- **Por comandos**: onde as requisições são feitas diretamente.
- **Por SSE (Server-Sent Events)**: onde a comunicação ocorre de maneira contínua e reativa.

Nesta implementação, utilizamos **FastMCP** para construir o servidor, e por padrão, ele roda no modo SSE.

Se você quiser ver exemplos de implementações de MCP, você pode ver esta [listagem de MCPs](https://mcp.so/) e caso você queira ler as especificações originais do protocolo, você pode ler a [documentação da Anthropic](http://anthropic.com/news/model-context-protocol).

## Configurando o ambiente

Antes de mais nada, precisamos garantir que temos tudo pronto para rodar o nosso servidor MCP.

### Instalando dependências
Para isso, utilizamos o **uv**, um gerenciador de pacotes super-rápido. Execute o seguinte comando para inicializar o projeto:

```bash
uv init
```

### Instalando o FastMCP

O **FastMCP** é uma biblioteca que simplifica a criação de servidores MCP. Para instalá-lo, execute:

```bash
uv add "mcp[cli]"
```

## Criando o Servidor MCP com FastMCP e mcp[cli]

Agora que temos o ambiente configurado, vamos criar um servidor MCP utilizando **FastMCP** e **mcp[cli]**.

Crie um arquivo `server.py` e adicione o seguinte código:

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("MyToolServer")

@mcp.tool()
def calculate(operation: str, a: float, b: float) -> float:
    """
    Perform a basic calculation.
    
    Args:
        operation: The operation to perform (add, subtract, multiply, divide)
        a: First number
        b: Second number
        
    Returns:
        The result of the calculation
    """
    if operation == "add":
        return a + b
    elif operation == "subtract":
        return a - b
    elif operation == "multiply":
        return a * b
    elif operation == "divide":
        if b == 0:
            raise ValueError("Cannot divide by zero")
        return a / b
    else:
        raise ValueError(f"Unknown operation: {operation}")

if __name__ == "__main__":
    # Run the server using Server-Sent Events protocol
    mcp.run("sse")
```

Esse código inicializa um servidor MCP pronto para aceitar conexões e define uma ferramenta simples de cálculo matemático.

## Rodando o servidor
Agora que temos tudo pronto, podemos iniciar o servidor MCP com o comando:

```bash
uv run server.py
```

Pronto! Nosso servidor já está rodando e pronto para receber conexões!

## Como conectar sua IDE ao MCP

Agora que temos o servidor em execução, podemos configurá-lo em uma IDE que suporte MCP, como **Cursor** ou **Zed**.

### Configuração do servidor na IDE
Para utilizar o servidor MCP, basta configurar as seguintes informações na sua IDE:

```
Nome do servidor: <Qualquer nome>
URL: http://localhost:8000/sse ou a URL do seu servidor com a rota /sse
```

Isso permitirá que sua IDE se conecte ao servidor MCP e utilize suas funcionalidades.

## Utilizando o MCP dentro do Cursor

Se você estiver usando o **Cursor**, pode acessar o MCP diretamente pelo **Composer Mode**, que fica na aba lateral direita, junto ao chat e ao bug finder. Isso permite interagir com as ferramentas MCP de maneira integrada e fluida.

## Conclusão

Parabéns! Agora você tem um servidor MCP rodando com **FastMCP** e pode utilizá-lo para interagir com APIs externas de forma eficiente. Isso abre um mundo de possibilidades para conectar seu LLM a diversas ferramentas externas, automatizando processos e ampliando o potencial das suas aplicações. 🚀

Agora é hora de testar, explorar e criar algo incrível com MCP! 💡

