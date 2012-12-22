Novidade no Symfony 2.2: Melhor interação no Console
====================================================

(Tradução de: http://symfony.com/blog/new-in-symfony-2-2-better-interaction-from-the-console)

O Symfony não é apenas um framework web. Ele também é um conjunto de componentes que solucionam os desafios enfrentados pelos desenvolvedores em seu dia-a-dia. O componente Console é um dos componentes do Symfony, não relacionados com a web, que ajudam você a criar belos programas de linha de comando com facilidade. No symfony 2.2, o componente Console foi melhorado, principalmente no que diz respeito à interação com os usuários.

Exibindo uma barra de progresso para as tarefas de longa execução
-----------------------------------------------------------------

Ao executar comandos de longa duração pela CLI, fornecer um feedback para o usuário, durante a execução de seu comando, é algo realmente útil. O Symfony 2.2 disponibiliza um helper de barra de progresso que faz todo o trabalho para você::

    $progress = $app->getHelperSet()->get('progress');

    $progress->start($output, 50);
    $i = 0;
    while ($i++ < 50) {
        // faz algo

        // avança a barra de progresso em 1 unidade
        $progress->advance();
    }

    $progress->finish();

E claro, tudo é personalizável, como explicado na `documentação`_.

Escondendo as senhas fornecidas na CLI
--------------------------------------

Se você quer pedir uma senha através da CLI, é melhor esconder o que o usuário digita. A partir do Symfony 2.2, existe uma forma conveniente para esconder o que os usuários digitam na linha de comando, através do novo método askHiddenResponse::

    $dialog      = $this->getHelperSet()->get('dialog');
    $password = $dialog->askHiddenResponse($output, 'Qual é a senha do banco de dados?');

Pedindo ao usuário para escolher a partir de uma lista de opções
----------------------------------------------------------------

Há muitas formas de pedir algumas informações na CLI. No Symfony 2.2, é possível até mesmo restringir o que o usuário pode digitar através do novo helper select(). O código a seguir demonstra algumas possibilidades que você tem agora::

    use Symfony\Component\Console\Application;
    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Output\OutputInterface;

    $app = new Application();
    $app->register('ask-color')->setCode(function (InputInterface $input, OutputInterface $output) use ($app) {
        $dialog = $app->getHelperSet()->get('dialog');
        $colors = array('vermelha', 'azul', 'amarela');

        // simplesmente pede uma cor (sem validação)
        $color = $dialog->ask($output, 'Digite a sua cor favorita (a cor padrão é vermelha): ', 'vermelha');
        $output->writeln('Você acabou de digitar: '.$color);

        // pede e valida a resposta
        $color = $dialog->askAndValidate($output, 'Digite a sua cor favorita (a cor padrão é vermelha): ', function ($color) use ($colors) {
            if (!in_array($color, array_values($colors))) {
                throw new \InvalidArgumentException(sprintf('A cor "%s" é inválida.', $color));
            }

            return $color;
        }, false, 'vermelha');
        $output->writeln('Você acabou de digitar: '.$color);

        // força o usuário a selecionar a partir de uma lista pré-definida de opções
        $color = $dialog->select($output, 'Por favor selecione a sua cor favorita (a cor padrão é vermelha)', $colors, 0);
        $output->writeln('Você acabou de selecionar: '.$colors[$color]);
    });

    $app->run();

.. _`documentação`: http://symfony.com/doc/master/components/console/introduction.html#displaying-a-progress-bar
