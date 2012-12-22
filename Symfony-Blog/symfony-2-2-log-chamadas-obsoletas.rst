Novidade no Symfony 2.2: Log das chamadas obsoletas
===================================================

(Tradução de http://symfony.com/blog/new-in-symfony-2-2-logging-of-deprecated-calls)

A primeira versão de Suporte a Longo Prazo (LTS) do Symfony2 será a 2.3, a ser lançada em Maio de 2013. 
Enquanto isso, estamos em uma corrida para finalizar e polir tudo, uma vez que, não vamos mais permitir quebras de compatibilidade com as versões anteriores, após a versão 2.3, sem uma razão muito boa (por exemplo,
uma questão de segurança). Assim, desde a versão 2.0, em vez de apenas substituir ou remover as funcionalidades existentes, nós tornamos elas obsoletas, e serão, então, removidas na versão 2.3.

Mas como você pode acompanhar essas funcionalidades obsoletas? Com certeza, os CHANGELOGs e o arquivo UPGRADE mencionam todas as quebras de compatibilidade com versões anteriores e as funcionalidades obsoletas. O próprio código também contém algumas dicas sobre os métodos e classes obsoletos pois eles são marcados com @deprecated.
Mas, e se você perder algumas ocorrências no seu código? Será que o seu código irá explodir quando atualizar para a versão 2.3? A resposta curta é sim.

Então, para fornecer uma transição mais suave, o Symfony 2.2 apresenta um novo recurso: o log das chamadas obsoletas. Sempre que você chamar um método obsoleto ou quando criar uma instância de uma classe obsoleta, o Symfony agora vai fazer o log da chamada e alertá-lo na barra de ferramentas de depuração. Também é possível verificar exatamente onde a chamada ocorreu no painel de log do profiler web.

E, se você tem alguns testes funcionais, ou se apenas navegar pela sua aplicação manualmente, poderá automatizar tudo analisando o arquivo de log do Symfony:

[2012-12-17 11:24:03] deprecation.WARNING: foo {
    "type":-100,
    "file":"/path/to/Controller/SomeController.php",
    "line":37,
    "stack":[/* stack trace not show here */]
}
