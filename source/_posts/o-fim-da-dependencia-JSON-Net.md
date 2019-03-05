---
title: .NET Core 3.0 (Preview 2) - O fim da dependência do JSON.NET
date: 2019-03-05 12:35:56
tags: 
- .Net Core 3.0
- C#
categories: 
- Programação
---
## O Por Quê

Não há o que se questionar sobre a eficiência do [JSON.NET](https://www.newtonsoft.com/json). O gráfico abaixo, coletado do próprio site, mostra a eficiência dele frente a outras ferramentas bastante utilizadas no mercado.

{% img /images/20190305/jsonperformance.png %}

Já trabalhei com outros, com ferramentas autorais, enfim, comprovadamente nenhuma obtivera um desempenho tão eficiente quando o JSON.NET.
O próprio ecossistema .NET dependia do JSON.NET e de outras bibliotecas populares – que ainda continuam sendo boas opções. Mas porque o .NET decide então findar a dependência do JSON.NET?

Podemos entender o porquê no próprio [github](https://github.com/dotnet/announcements/issues/90) do dotnet:

•	Foi preciso uma API de JSON de alto desempenho, bem maior do que JSON.NET e que dê suporte, também, ao novo tipo [Span&lt;T&gt;](https://imasters.com.br/back-end/novidades-do-c-7-2-parte-03-read-only-structs-e-o-tipo-span) - que foi projetada para performar o gerenciamento e operações de escrita em regiões contíguas de memória. O JSON.NET é implementado lendo UTF-16 puramente por motivos de análise. A capacidade de ler e gravar documentos em UTF-8 era necessário.

•	O JSON.NET faz parte do ASP.NET Core. Ao longo dos anos foi percebido que esta dependência pode entrar em conflito com outras bibliotecas que tem sua própria dependência de versão diferente do JSON.NET.

Por estas razões, o ecossistema .NET Core passa a ter suporte interno a um JSON de alto desempenho, baixa alocação e baseada em Span<bytes>. É uma implementação que está num nível mais baixo e por isso obtem um desempenho melhor.

Vamos testar alguns cenários para comprovar. A idéia é confrontar o tempo de execução com o Json.NET.

## Utf8JsonWriter

Vocês precisaram baixar o [Visual Studio 2019 Preview](https://visualstudio.microsoft.com/pt-br/vs/preview/) para conseguirem acompanhar, por causa da compatibilidade da versão da linguagem e do framework, C# 8 e .Net Core 3.0 preview 2.

{% img /images/20190305/vs2019Preview.png %}

Abra o terminal e digite: 
{% codeblock lang:Batch %}
dotnet new blazor -o EstudoCSharp8Json
cd EstudoCSharp8Json
dotnet run
{% endcodeblock %}

A classe Utf8JsonWriter precisa da interface IBufferWriter&lt;byte&gt; como localização de saída na qual os dados são gravados, mas atualmente o framework ainda não possui uma implementação concreta. Para o nosso teste eu obtive a classe [IBufferWriter&lt;byte&gt;](https://gist.github.com/ahsonkhan/c76a1cc4dc7107537c3fdc0079a68b35). Crie uma classe chamada ArrayBufferWriter e copie o código abaixo:

{% codeblock lang:csharp %}
public class ArrayBufferWriter : IBufferWriter<byte>, IDisposable
{
    private byte[] _rentedBuffer;
    private int _written;
    private long _committed;

    private const int MinimumBufferSize = 256;

    public ArrayBufferWriter(int initialCapacity = MinimumBufferSize)
    {
        if (initialCapacity <= 0)
            throw new ArgumentException(nameof(initialCapacity));

        _rentedBuffer = ArrayPool<byte>.Shared.Rent(initialCapacity);
        _written = 0;
        _committed = 0;
    }

    public Memory<byte> OutputAsMemory
    {
        get
        {
            CheckIfDisposed();

            return _rentedBuffer.AsMemory(0, _written);
        }
    }

    public Span<byte> OutputAsSpan
    {
        get
        {
            CheckIfDisposed();

            return _rentedBuffer.AsSpan(0, _written);
        }
    }

    public int BytesWritten
    {
        get
        {
            CheckIfDisposed();

            return _written;
        }
    }

    public long BytesCommitted
    {
        get
        {
            CheckIfDisposed();

            return _committed;
        }
    }

    public void Clear()
    {
        CheckIfDisposed();

        ClearHelper();
    }

    private void ClearHelper()
    {
        _rentedBuffer.AsSpan(0, _written).Clear();
        _written = 0;
    }

    public async Task CopyToAsync(Stream stream, CancellationToken cancellationToken = default)
    {
        CheckIfDisposed();

        if (stream == null)
            throw new ArgumentNullException(nameof(stream));

        await stream.WriteAsync(_rentedBuffer, 0, _written, cancellationToken).ConfigureAwait(false);
        _committed += _written;

        ClearHelper();
    }

    public void CopyTo(Stream stream)
    {
        CheckIfDisposed();

        if (stream == null)
            throw new ArgumentNullException(nameof(stream));

        stream.Write(_rentedBuffer, 0, _written);
        _committed += _written;

        ClearHelper();
    }

    public void Advance(int count)
    {
        CheckIfDisposed();

        if (count < 0)
            throw new ArgumentException(nameof(count));

        if (_written > _rentedBuffer.Length - count)
            throw new InvalidOperationException("Cannot advance past the end of the buffer.");

        _written += count;
    }

    // Returns the rented buffer back to the pool
    public void Dispose()
    {
        if (_rentedBuffer == null)
        {
            return;
        }

        ArrayPool<byte>.Shared.Return(_rentedBuffer, clearArray: true);
        _rentedBuffer = null;
        _written = 0;
    }

    private void CheckIfDisposed()
    {
        if (_rentedBuffer == null)
            throw new ObjectDisposedException(nameof(ArrayBufferWriter));
    }

    public Memory<byte> GetMemory(int sizeHint = 0)
    {
        CheckIfDisposed();

        if (sizeHint < 0)
            throw new ArgumentException(nameof(sizeHint));

        CheckAndResizeBuffer(sizeHint);
        return _rentedBuffer.AsMemory(_written);
    }

    public Span<byte> GetSpan(int sizeHint = 0)
    {
        CheckIfDisposed();

        if (sizeHint < 0)
            throw new ArgumentException(nameof(sizeHint));

        CheckAndResizeBuffer(sizeHint);
        return _rentedBuffer.AsSpan(_written);
    }

    private void CheckAndResizeBuffer(int sizeHint)
    {
        if (sizeHint == 0)
        {
            sizeHint = MinimumBufferSize;
        }

        int availableSpace = _rentedBuffer.Length - _written;

        if (sizeHint > availableSpace)
        {
            int growBy = sizeHint > _rentedBuffer.Length ? sizeHint : _rentedBuffer.Length;

            int newSize = checked(_rentedBuffer.Length + growBy);

            byte[] oldBuffer = _rentedBuffer;

            _rentedBuffer = ArrayPool<byte>.Shared.Rent(newSize);


            oldBuffer.AsSpan(0, _written).CopyTo(_rentedBuffer);
            ArrayPool<byte>.Shared.Return(oldBuffer, clearArray: true);
        }
    }
}
{% endcodeblock %}

Crie uma classe Pessoa.cs:
{% codeblock lang:csharp %}
public class Pessoa
{
    public int Id { get; set; }
    public string Nome { get; set; }

    public string ToJson()
    {
        var output = new ArrayBufferWriter();
        var json = new Utf8JsonWriter(output, state: default);
        json.WriteStartObject();
        json.WriteString("id", Id.ToString());
        json.WriteString("nome", Nome);
        json.WriteEndObject();
        json.Flush();
        return Encoding.UTF8.GetString(output.OutputAsMemory.Span);
    }

    public string ToJsonJsonNet() => JsonConvert.SerializeObject(this);
}
{% endcodeblock %}

Adicionem os namespaces sugeridos pelo VS2019. Se estiver tudo certo, você vai conseguir compilar a solução sem erros. Ressalto que o namespace System.Text.Json está preso ao framework e linguagem.

No método Main do arquivo Program.cs altere para o código abaixo:

{% codeblock lang:csharp %}
static void Main(string[] args)
{
    var pessoa = new Pessoa { Id = 1, Nome = "D. Pedro I" };
    var sw = new Stopwatch();
    sw.Start();
    var json = pessoa.ToJson();
    sw.Stop();
    Console.WriteLine($"Json: {sw.Elapsed.TotalSeconds}");
    var sw2 = new Stopwatch();
    sw2.Start();
    var jsonNet = pessoa.ToJsonJsonNet();
    sw2.Stop();
    Console.WriteLine($"Json.Net: {sw2.Elapsed.TotalSeconds}");
    Console.ReadKey();
}
{% endcodeblock %}

Vejam o resultado abaixo e obtenham a própria conclusão:

{% img /images/20190305/JsonWriter.PNG %}

Exatamente 0,7936707s / 0,1891906s = 4,195085273792673 vezes mais rápido. Claro que estamos trabalhando com uma classe muito pequena, mas confirma o resultado vendido.

## JsonDocument

Vamos utilizar o json gerado pelo nosso experimento anterior para realizar a desserialização, ou seja, converter o texto json para o objeto pessoa e confrontar também com o Json.NET.

Adicione os métodos abaixo na classe Pessoa.cs:

{% codeblock lang:csharp %}
public static Pessoa ToPessoa(string json)
{
    var pessoa = new Pessoa();
    using (JsonDocument doc = JsonDocument.Parse(json))
    {
        JsonElement root = doc.RootElement;
        pessoa.Id = Convert.ToInt32(root.GetProperty("id").GetString());
        pessoa.Nome = root.GetProperty("nome").GetString();
    }
    return pessoa;
}

public static Pessoa ToJsonNet(string json) => JsonConvert.DeserializeObject<Pessoa>(json);
{% endcodeblock %}

Modifique o método principal Main para como está abaixo:

{% codeblock lang:csharp %}
static void Main(string[] args)
{
    var json = "{\"id\":\"1\",\"nome\":\"D. Pedro I\"}";
    var sw = new Stopwatch();
    sw.Start();
    var pessoa = Pessoa.ToPessoa(json);
    sw.Stop();
    Console.WriteLine($"Json: {sw.Elapsed.TotalSeconds}");
    var sw2 = new Stopwatch();
    sw2.Start();
    var pessoa2 = Pessoa.ToJsonNet(json);
    sw2.Stop();
    Console.WriteLine($"Json.Net: {sw2.Elapsed.TotalSeconds}");
    Console.ReadKey();
}
{% endcodeblock %}

Vamos ver o resultado do confronto:

{% img /images/20190305/JsonDocument.PNG %}

O pacote do .Net Core é simplesmente 0,6848968s / 0,0559926s = 12,23191636037619 vezes mais rápido do que o Json.Net. Impressionante. Mesmo realizando alguns testes a mais o resultado tornou-se sempre similar. Claro que estamos trabalhando com uma classe muito pequena, mas a tendência é que o resultado se comprove, pela própria argumentação da Micrsoft. Estou convencido.

Existe ainda a classe Utf8JsonReader, mas a classe JsonDocument já embasada nela, por isso não a abordarei neste artigo, mas nas referências abaixo, você poderá consultá-la.

Como falamos acima, agora comprovado, além do suporte ao tipo especial Span&lt;T&gt;, a velocidade na serialização e desserialização, fora problemas de compatibilidade com versões do Json.NET, me parece que o .Net Core, aqui, tem seguido fielmente o objetivo.

Até a próxima senhores!



## Referências
- https://docs.microsoft.com/pt-br/dotnet/core/whats-new/dotnet-core-3-0#main
- https://github.com/dotnet/announcements/issues/90
- https://www.newtonsoft.com/json
- https://imasters.com.br/back-end/novidades-do-c-7-2-parte-03-read-only-structs-e-o-tipo-span




