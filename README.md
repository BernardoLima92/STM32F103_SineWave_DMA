# STM32F103_SineWave_DMA
Geração de um Sinal Senoidal utilizando o STM32F103C8T6 (Bluepill)

Este tutorial descreve o passo a passo para a geração de um Sinal Senoidal usando o STM32F103C8T6 (Bluepill), utilizando o software STMCubeIDE. 
Para a criação do sinal senoidal serão utilizados os Timers 2 e 4 do microcontrolador. Além disso, também será necessário ter previamente uma tabela com o 
sinal a ser gerado.

Como o STM32F103C8T6 (Bluepill) não possui um DAC interno, será utilizada uma porta PWM para simular um conversor digital analógico (DAC). O Timer 2 será 
utilizado para gerar o sinal PWM. O Timer 4 será utilizado para atualizar periodicamente o valor do duty cycle do sinal PWM gerado pelo Timer 2.

A Figura abaixo explica como ocorre a geração do sinal.
![Diagrama SineWave](https://user-images.githubusercontent.com/114233216/192160554-a5d5b16f-46d8-4578-95d8-257733cf1697.png)

O timer 2 é configurado para trabalhar no modo PWM Generation e utiliza os registradores TIM2_CNT e TIM2_CCR1 para controlar o valor do duty cycle. 
O valor de CNT inicia em 0 e é incrementado em 1 a cada pulso de clock até um valor máximo determinado pelo registrador ARR. Após CNT se igualar a 
ARR o valor de CNT é zerado e toda contagem reinicia novamente. Quando o valor de TIM2_CNT < TIM2_CCR1, a saída A0 do STM32 assume nível lógico alto. 
Quando TIM2_CNT > TIM2_CCR1 a saída A0 assume nível lógico baixo. Escolhendo de forma adequada o valor de TIM2_CNT e TIM2_CCR1 é possível escolher um
valor analógico com base no duty cycle. Na figura 1 letra a), o Timer2 conta de 0 até 255. Para fins de exemplo, foi escolhido um valor de TIM2_CCR1 
igual a 80. O resultado na saída da porta PWM pode ser visto na letra c). Quando TIM2_CNT <= 80, o valor da saída A0 é nível lógico alto, quando 
TIM2_CNT > 80, o valor de A0 passa a ter um nível lógico baixo.

O Timer4 usa os registradores TIM4_CNT e TIM4_ARR para especificar os instantes de tempo no qual um dado é transferido da tabela de senos para o registrador 
TIM2_CCR1 do Timer2, para que o sinal PWM seja gerado. O valor de TIM4_CNT inicia em 0 e é incrementado em 1 a cada pulso de clock até um valor máximo 
determinado pelo registrador TIM4_ARR. Após TIM4_CNT se igualar a TIM4_ARR ocorre um Update Event e o valor de TIM4_CNT é zerado e toda contagem reinicia 
novamente. No Update Event é gerada uma requisição DMA para que uma WORD (32 bits) seja transferida da memória (Tabela de Senos) para o registrador TIM2_CCR1. 
Essa informação contém o novo valor que o registrador TIM2_CCR1 irá assumir. Na figura 1 letra c), o Timer4 conta de 0 até 768. Nesse momento ocorre um 
Update Event e uma requisição DMA é gerada para atualizar o valor de TIM2_CCR1. Um ponto importante a ser observado é que essa tarefa de atualizar o valor 
de TIM2_CCR1 é realizada via DMA, fato que deixa o núcleo totalmente livre para realizar outras atividades.

Como calcular a frequência do sinal a ser gerado?

Para se calcular a frequência do sinal gerado são utilizadas 3 equações:
![equações](https://user-images.githubusercontent.com/114233216/192160645-9b8ecff7-593b-408d-a0e8-9d30efc07814.png)

A equação 1 determina o valor da frequência do sinal PWM gerado pelo Timer 2. 
O valor Fclk é o valor do clock que alimenta o timer. Por padrão, o STM32F103C8T6 possui um clock de 8MHz fornecido por um cristal presente na placa. 
O Valor ARR determina o final da contagem. O valor PSC (Prescaler) é um fator de escala que pode ser adicionado ou não à frequência principal Fclk do timer. 

A equação 2 determina o valor da frequência de trigger, isto é, a frequência com que se atualiza o valor de TIM2_CCR1. É a frequência com que o DMA 
transfere uma WORD da memória para o registrador TIM2_CCR1 do timer 2.

A equação 3 determina a frequência do sinal gerado. Ns é o tamanho do sinal, ou seja, o número total de pontos utilizados para se criar a tabela com o sinal.

Detalhe importante é que a frequência do sinal PWM deve ser maior (pelo menos 2x) do que a frequência de trigger.

Exemplo:
Flck = 72 MHz	// Isso é possível utilizando o PLL disponível no STM32.

TIM2_ARR = 255
TIM2_ PSC = 0
Fpwm = 281250 Hz = 281KHz

TIM4_ARR = 768
TIM4_PSC = 0
Ftrigger = 93750 Hz = 93KHz

Ns = 128
FSinal = 732,4 Hz = 732 Hz

As etapas a seguir mostram a parte prática:

1. Criação da Tabela no Excel e em Python
2. Configuração do Clock
3. Configuração do Timer 2
4. Configuração do Timer 4 + DMA
5. Resultado Obtido 


1. Criação da Tabela no Excel e em Python

Para a geração do sinal desejado é preciso ter previamente uma tabela contendo os valores discretos de cada ponto do sinal a ser gerado. 
Neste exemplo vamos gerar um sinal senoidal discreto com um total de 128 pontos. Além disso, o sinal gerado vai ter uma resolução de 8 bits, 
isto é, sua amplitude máxima será representada por 255. Em termos práticos, quando a porta PWM receber o valor 255, sua saída será 3,3V e 
quando a porta PWM receber o valor 0, sua saída será 0V.
![Tabela_DDS](https://user-images.githubusercontent.com/114233216/192160707-1b334160-9eff-4b85-a328-6bb45cb4029b.png
O primeiro valor da tabela (Ponto N° 0) é igual a 127. O segundo valor (Ponto N° 1) é igual a 133. E assim por diante, até o Ponto N° 127, 
que é a última linha da tabela. Esses valores foram determinados pela fórmula:
![Fórmula](https://user-images.githubusercontent.com/114233216/192160745-8e063830-69fe-48a2-b6b3-b8c5bf31eeb8.png)
=(ARRED(((((2^C$3)/2)-1)*SEN(2*PI()*B7/C$4));0))+(((2^C$3)/2)-1)

O resultado é um gráfico de uma função senoidal, com 128 pontos, cujos valores variam entre 0 e 255. Observação importante é que todos 
os valores da tabela são positivos, isso significa que o valor da função foi normalizado.

Para a geração do sinal em python foi usada a seguinte rotina:
![Python](https://user-images.githubusercontent.com/114233216/192160757-3eff0035-7fb2-4252-811f-434d75f2a481.png)

A resolução do sinal PWM gerado é similar à resolução de um DAC. Por exemplo, um DAC de 10 bits produz um sinal mais suave do que um DAC de 8 bits. 
A figura abaixo mostra esse mesmo sinal senoidal com 128 amostras com uma resolução de 5 bits. 
![seno5bits](https://user-images.githubusercontent.com/114233216/192160775-32dc912d-2798-4c66-b495-8475d459238b.png)

Logo, a conclusão que se tira é que há uma relação entre resolução e tamanho do sinal. É preciso testar até encontrar um valor ótimo. 
Em teoria, quanto maior a resolução e maior o tamanho do sinal, melhor ele será. Porém, como vimos anteriormente, estas variáveis interferem 
no valor da frequência do sinal gerado.

Aqui nesse exemplo foi gerado um sinal senoidal. Contudo, qualquer forma de onda pode ser gerada, basta que se crie uma tabela do jeito desejado.


2. Configuração do CLock
![TIM4_Clock](https://user-images.githubusercontent.com/114233216/192160839-e626d379-0b42-4f8d-9897-aa0b39ca3f99.png)

3. Configuração do TIMER 2
![TIM2_fig1](https://user-images.githubusercontent.com/114233216/192160854-1843bdca-6cde-4c16-bc79-9cc81784d45f.png)
![TIM2_fig2](https://user-images.githubusercontent.com/114233216/192160855-9d547738-f960-4023-a231-7907fe14e3de.png)

4. Configuraçõ do Timer 4 + DMA
![TIM4_fig1](https://user-images.githubusercontent.com/114233216/192160876-79a15042-b8f5-46ce-9d45-4ea7e93f7b7e.png)
![TIM4_fig2](https://user-images.githubusercontent.com/114233216/192160877-2ed1878e-6c1b-4ed1-993e-ec57b9bed617.png)
![TIM4_fig3](https://user-images.githubusercontent.com/114233216/192160873-720dc607-61b0-497d-8714-3aec2d90ed77.png)


5. Resultado Obtido

![1](https://user-images.githubusercontent.com/114233216/192160897-0485f3aa-0e54-4624-b7fe-b1c4faea074f.png)

6. Parte Principal do Código
![código](https://user-images.githubusercontent.com/114233216/192160934-1697cbe6-bd1a-4200-95f9-ed0b84e40645.png)












