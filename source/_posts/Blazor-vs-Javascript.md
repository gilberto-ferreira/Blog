---
title: .NET Core 3.0 >> Blazor vs Javascript, Desempenho nas Requisições XHR
tags:
- Blazor
- .Net Core 3.0
- Web Assembly
- C#
- SPA
categories: 
- Programação
---

[Blazor](https://blazor.net/) é um framework da Microsoft para o desenvolvimento de aplicativos [SPA](https://imasters.com.br/desenvolvimento/single-page-applications-e-outras-maravilhas-da-web-moderna). Mas como fazer o navegador entender o código .NET?

Utilizando o [WebAssembly](https://webassembly.org/), uma tecnologia experimental, que já é suportada pelos principais navegadores. Sua estrutura foi otmizada para downlods e velocidade de execução. É um formato de bytecode que só pode fazer o que o Javascript faz.

Não será nosso foco entrar em questões elementares do web assembly. No final do artigo deixarei referências para consulta.

Quem já está acostumado com [Razor](https://docs.microsoft.com/pt-br/aspnet/core/razor-pages/?view=aspnetcore-2.2&tabs=visual-studio) não vai sentir sequer dificuldade, pois o Blazor me parece ser uma evolução do Razor.

Neste artigo vamos montar um exmplo que faz uma chamada http GET via javascript puro e via blazor e vamos comparar usando as ferramentas de depuração do navegador Google Chrome. Os fontes estudados neste post podem ser consultados neste link.

Antes de iniciar você precisará instalar a versão 3.0 (preview) do .NET Core. Acesse este [link](https://docs.microsoft.com/pt-br/aspnet/core/client-side/spa/blazor/get-started?view=aspnetcore-3.0&tabs=netcore-cli&viewFallbackFrom=aspnetcore-2.1) para ver o passo a passo para criar o projeto. Caso possua alguma dificuldade para levantar o projeto blazor, veja esta [referência](https://github.com/dotnet/sdk/issues/2948).

Abra o seu terminal e insira o código abaixo:
{% codeblock lang:Batch %}
dotnet new blazor -o BlazorTestRest
cd BlazorTestRest
dotnet run
{% endcodeblock %}

Você deverá ver algo do tipo em seu terminal:

{% img /images/20190303/dotnetrun.jpg %}

Abra o seu navegador e vá até o link [fetchdata](http://localhost:5000/fetchdata). Você deverá estar na tela abaixo.

{% img /images/20190303/tela.jpg %}

O template padrão de um projeto Blazor já faz uma requisição XHR. Nós faremos apenas uma adaptação. Iremos carregar os dados listados na tela ao acionar um botão. Limpe o html e a função existente para o carregamento da página e adicione o código abaixo dentro do bloco <i>funcions</i> do arquivo. A ideia é isolar o cenário.

{% codeblock lang:csharp %}
public async void LoadForecast() 
{
    Console.WriteLine("Evento de click engatilhado");
    forecasts = await Http.GetJsonAsync<WeatherForecast[]>("sample-data/weather.json");
    Console.WriteLine("Evento de click finalizado");
}
{% endcodeblock %}

Agora adicione o botão de acordo com o código HTML abaixo:
{% codeblock lang:html %}
<button onclick="@LoadForecast">Load Forecasts</button>
{% endcodeblock %}

Se tudo estiver certo, seu código deverá estar assim:

{% img /images/20190303/codigo.jpg %}

Assim como eu, quem já está habituado a trabalhar com Razor, a dificuldade foi zero! Clique no botão e acompanhe o resultado pelas ferramentas de network e console do Chrome. Force um erro de conexão e clique no botão - veja a descrição dos erros que são inscritos no console.

Vamos implementar a mesma chamada usando javascript puro e comparar os resultados. Crie um site, html puro mesmo, e adicione o código abaixo:

{% codeblock lang:html %}
<div id="appTeste">
	<button onclick="getForecasts()">Load Forecasts</button>
</div>
<script>
	getForecasts = function() {
		console.log('Iniciado');
		fetch("http://localhost/TesteGet/weather.json")
			.then(function(result) { 
				result.text()
					.then(function(x) { 
                        var forecasts = JSON.parse(x);
                        console.log('Terminado'); 
                    });
			});
	}
</script>
{% endcodeblock %}

Esse código é muito simples e faz basicamente o que o outro está fazendo. Queremos apenas testar o desempenho de cada um.

## Parciais
| # | Blazor | Javascript |
| - | - |
| Velocidade | 10.65ms | 6.3ms |
| Peso do Arquivo | 512B | 828B |

Estes critérios são totalmente objetivos, então é preciso atentar para o que custa pra mais pra você, peso ou velocidade. Devo ressaltar que foi utilizado a média aritimética para 20 requisições para cada opção.

Vamos agora considerar um dataset maior e ver qual tecnologia tem o melhor desempenho. Copiei o conteúdo [deste](https://blockchain.info/unconfirmed-transactions?format=json) dataset para dentro do dataset weather. É um dataset que contem informações de tamanho mediano e bastante comum também de ser utilizado.

## Parciais com Dataset Mediano
| # | Blazor | Javascript |
| - | - |
| Velocidade | 9.3ms | 7.85ms |
| Peso do Arquivo | 6.5KB | 19.8KB |

Curiosamente as requisições com Blazor obtiveram um desempenho com um arquivo com peso mediano do que com um arquivo muito pequeno. De igual forma, javascript puro ainda obteve um melhor desempenho em velocidade e Blazor no peso.

## Parciais com Dataset Grande
| # | Blazor | Javascript |
| - | - |
| Velocidade | 317.8ms | 217.45ms |
| Peso do Arquivo | 2.4MB | 9.3MB |

Considerando um [dataset](http://www.vizgr.org/historical-events/search.php?format=json&begin_date=-3000000&end_date=20151231&lang=en) grande, continuamos com o mesmo comportamento. Blazor lida melhor com o tamanho dos arquivos, mas demora mais pra responder com um arquivo menor. É importante dizer que com este arquivo maior, senti um incômodo no navegador, como se ele estivesse "agarrado", ao realizar as requisções pelo blazor. Na medida que eu clico sequencialmente, as requisições são mais pesadas, enquanto quando eu espero a página liberar, elas são mais leves.

Quis focar no desempenho do blazor neste arquivo porque tenho observado a comunidade bastante entusiasmada. Particularmente tenho aprendido a ter muito receio ao escolher uma tecnologia. Algumas bibliotecas, mesmo da própria Microsoft, podem ser muito mal empregadas por falta de conhecimento - o caso Entity Framework e OData, da Microsoft mesmo.

Esta foi uma introdução ao Blazor. No próximo artigo vamos nos aprofundar no framework criando uma SPA com as funções básicas: como um CRUD, roteamento, como é simples registrar um web component, dentre outras funcionalidades. Vou me esforçar para colocar tudo em um artigo. 

Até a próxima!

## Referências
- [Web Assembly](https://webassembly.org/)
- [Razor](https://docs.microsoft.com/pt-br/aspnet/core/razor-pages/?view=aspnetcore-2.2&tabs=visual-studio)
- [Blazor](http://learn-blazor.com)
- [Single Page Applications e outras maravilhas da web moderna](https://imasters.com.br/desenvolvimento/single-page-applications-e-outras-maravilhas-da-web-moderna)
- [Introução ao Blazor](https://docs.microsoft.com/pt-br/aspnet/core/client-side/spa/blazor/get-started?view=aspnetcore-3.0&tabs=netcore-cli&viewFallbackFrom=aspnetcore-2.1)
- [Erro Unable to generate deps.json](https://github.com/dotnet/sdk/issues/2948)
