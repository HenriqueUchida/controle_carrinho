# Controle Web para Hockey Bot

Controle remoto autoral, feito para rodar no navegador do celular, para o carrinho **Hockey Bot** da RoboCore.

O projeto foi desenvolvido para a **Semana de Engenharia do Centro Universitário Senac**.

**Acesse:** https://henriqueuchida.github.io/controle_carrinho/

> Use o **Chrome no Android** ou o navegador **Bluefy no iPhone**, com o celular na horizontal. A página pede acesso ao Bluetooth quando você toca em **Conectar**.

## Sobre o projeto

O Hockey Bot já vem com um aplicativo oficial, e o protocolo de comunicação dele não é documentado. Para construir um controle próprio sem alterar o carrinho, foi feita **engenharia reversa da comunicação Bluetooth**: observar o que o aplicativo oficial envia ao carrinho e reproduzir esse comportamento em uma aplicação web.

O firmware do carrinho **não foi modificado**. O controle conversa com ele exatamente como o aplicativo original.

## Como a engenharia reversa foi feita

1. **Reconhecimento com o nRF Connect:** conexão ao carrinho para listar os serviços e as características Bluetooth Low Energy (BLE) que ele expõe.
2. **Captura de tráfego:** com o registro HCI do Android ativado, o aplicativo oficial foi usado em movimentos isolados (frente, trás, direita, esquerda), e cada sessão gerou um arquivo de log.
3. **Análise no Wireshark:** filtrando o protocolo `btatt`, foram isolados os pacotes enviados ao carrinho, convertidos de hexadecimal para texto e relacionados a cada movimento.
4. **Descoberta da etapa de ativação:** comparando a conexão completa do aplicativo oficial com os testes manuais, descobriu-se que o carrinho só aceita comandos de movimento depois que o cliente liga as notificações de bateria.
5. **Implementação e validação:** o protocolo foi reproduzido com a Web Bluetooth API e testado diretamente no carrinho.

## Protocolo descoberto

Serviço BLE customizado do carrinho:

| Item | UUID |
|---|---|
| Serviço | `dacabf1f-5f2e-4d16-b8f8-13bbaaec1349` |
| Característica de comando (Write Without Response) | `dacabf1f-5f2e-4d16-b8f8-13bbaaec5781` |

**Sequência ao conectar**

1. Ligar as notificações do serviço de bateria padrão (`0x180F`): *Battery Level* e *Battery Level Status*.
2. Enviar o comando de parada `0,0` duas vezes.
3. Passar a enviar comandos de movimento continuamente.

**Formato dos comandos**

Texto ASCII no formato `velocidade,ângulo`, enviado a cada 50 ms enquanto o joystick está em uso.

| Movimento | Comando |
|---|---|
| Frente | `100,90` |
| Trás | `100,270` |
| Esquerda | `100,180` |
| Direita | `100,360` |
| Parada | `0,0` |

A velocidade vai de 0 a 100, e o ângulo é medido em graus (90 = frente, 270 = trás). Ângulos intermediários produzem curvas, e a direita usa **360**, porque o carrinho não responde a `0` como ângulo.

## Funcionalidades

- Dois joysticks virtuais, como no aplicativo original: o da esquerda controla avanço e recuo, e o da direita controla a direção.
- Combinação dos dois eixos em um único comando de velocidade e ângulo.
- Tela cheia e orientação horizontal.
- Leitura do nível de bateria do carrinho.
- Compatível com qualquer Hockey Bot, sem depender do nome do dispositivo (que pode ser alterado no aplicativo oficial).

## Tecnologias

- [Vue 3](https://vuejs.org/) (`<script setup>`) com [Vite](https://vite.dev/)
- [Tailwind CSS](https://tailwindcss.com/)
- [Web Bluetooth API](https://developer.mozilla.org/docs/Web/API/Web_Bluetooth_API)
- GitHub Pages, com deploy automático pelo GitHub Actions

## Compatibilidade

| Ambiente | Funciona |
|---|---|
| Chrome ou Edge no Android | Sim |
| iPhone e iPad com o navegador **Bluefy** | Sim, pelo Bluefy (ainda não testado com o carrinho) |
| iPhone e iPad com Safari ou Chrome | Não, pois o iOS não suporta Web Bluetooth nesses navegadores |
| Chrome ou Edge no computador | Sim |

**No iPhone**, instale o [Bluefy](https://apps.apple.com/app/bluefy-web-ble-browser/id1492822055) na App Store e abra o endereço do projeto por ele. O Bluefy é um navegador gratuito que adiciona suporte à Web Bluetooth no iOS. Nele, a tela cheia e o travamento da rotação podem não funcionar. Se isso acontecer, basta girar o celular manualmente para a horizontal.

Só um aplicativo pode se conectar ao carrinho por vez. Feche o aplicativo da RoboCore antes de conectar.

## Estrutura

```
src/
  components/
    HockeyBotController.vue   interface, Bluetooth e protocolo
public/
  senac-logo.png              logo exibida no centro da tela
.github/workflows/
  deploy.yml                  publicação no GitHub Pages
```

## Avisos

- Este é um projeto educacional e independente. Ele **não é afiliado, patrocinado nem endossado pela RoboCore**. *Hockey Bot* e *RoboCore* são marcas de seus respectivos proprietários.
- A análise se limitou à observação do tráfego Bluetooth do carrinho usado no projeto, para fins de interoperabilidade e aprendizado.
- O nome e a logo do Senac pertencem à instituição.

## Créditos

Desenvolvido por [Henrique Uchida](https://github.com/HenriqueUchida) para a Semana de Engenharia do Centro Universitário Senac.