Novidade no Symfony 2.2: Utilitários de Segurança
=================================================

(Tradução de http://symfony.com/blog/new-in-symfony-2-2-security-utilities)

A partir do Symfony 2.2, refatoramos alguns utilitários de segurança para que você possa usá-los em seu próprio código. Estes utilitários estão disponíveis no namespace Symfony\Component\Security\Core\Util.

Gerando um número aleatório seguro
----------------------------------

Se você precisa gerar um número aleatório seguro, é melhor contar com uma implementação robusta. O Symfony fornece uma::

    use Symfony\Component\Security\Core\Util\SecureRandom;

    $generator = new SecureRandom();
    $random    = $generator->nextBytes(10);

O método nextBytes() retorna uma string aleatória composta pelo número de caracteres passados ​​como argumento (10 no exemplo acima).

Comparando Strings
------------------

`Timing attacks`_ não são bem conhecidos, mas ainda assim, o Symfony possui proteção contra eles. No Symfony 2.0 e 2.1, esta proteção foi aplicada para comparações de senha feitas no bundle Security, mas, a partir do Symfony 2.2, também está disponível para o desenvolvedor::

    use Symfony\Component\Security\Core\Util\StringUtils;

    // a senha1 é igual a senha2?
    $bool = StringUtils::equals($senha1, $senha2);

Quer aprender mais? Consulte a documentação em http://symfony.com/doc/master/book/security.html#utilities.

.. _`Timing attacks`: http://en.wikipedia.org/wiki/Timing_attack
