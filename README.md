#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

typedef enum {
    ESTADO_MENU,
    ESTADO_CREDITOS,
    ESTADO_SAIR,
    ESTADO_VILA,
    ESTADO_FASE1,
    ESTADO_FASE2,
    ESTADO_FASE3,
    ESTADO_VITORIA,
    ESTADO_DERROTA
} EstadoJogo;

#define JOGADOR '&'
#define PERSONAGEM 'P'
#define PAREDE '*'
#define CHAVE '@'
#define PORTA_FECHADA 'D'
#define PORTA_ABERTA '='
#define BOTAO 'O'
#define ESPINHO '#'
#define TELETRANSPORTE '>'
#define MONSTRO_NIVEL1 'X' // Monstro mais fraquinho
#define MONSTRO_NIVEL2 'V' // Monstro mais espertinho
#define VAZIO ' '

#define TAMANHO_VILA 10
#define TAMANHO_FASE1 10
#define TAMANHO_FASE2 20
#define TAMANHO_FASE3 40
#define TAMANHO_MAXIMO_MAPA 40

#define MAX_MONSTROS 10
#define MAX_TELETRANSPORTES 4
#define MAX_PERSONAGENS 5

// Estrutura para guardar informacoes dos personagens nao jogaveis (NPCs)
typedef struct {
    int linha;
    int coluna;
    int ativo; // 1 se o personagem esta no mapa, 0 caso contrario
    char mensagem[100]; // O que o personagem vai dizer
} Personagem;

// Estrutura para os monstros
typedef struct {
    int linha;
    int coluna;
    int ativo; // 1 se o monstro esta vivo, 0 se ja era
    int tipo;  // 0 para Nivel 1, 1 para Nivel 2
} Monstro;

EstadoJogo estadoAtual = ESTADO_MENU; // Comeca no menu principal
int jogoRodando = 1; // Enquanto for 1, o jogo continua
int contagemMortes = 0; // Quantas vezes o jogador bateu as botas na fase atual
int temChave = 0; // 1 se o jogador pegou a chave, 0 se nao
int linhaJogador, colunaJogador; // Onde o jogador esta no mapa

// Mapas de cada area do jogo
char mapaVila[TAMANHO_VILA][TAMANHO_VILA];
char mapaFase1[TAMANHO_FASE1][TAMANHO_FASE1];
char mapaFase2[TAMANHO_FASE2][TAMANHO_FASE2];
char mapaFase3[TAMANHO_FASE3][TAMANHO_FASE3];
char mapaBase[TAMANHO_MAXIMO_MAPA][TAMANHO_MAXIMO_MAPA]; // Guarda o estado original do mapa (sem o jogador/monstros se movendo)

Personagem personagens[MAX_PERSONAGENS]; // Vetor para guardar os NPCs
Monstro monstros[MAX_MONSTROS]; // Vetor para os monstros
int posicoesTeletransporte[MAX_TELETRANSPORTES][2]; // Guarda as coordenadas dos teletransportes (entrada e saida)
int numTeletransportes = 0; // Quantos teletransportes existem na fase
int botaoAtivo = 0; // 1 se o botao foi pressionado, 0 caso contrario

// --- Prototipos das Funcoes --- //
void inicializarJogo(void); // Prepara tudo pra comecar
void executarJogo(void); // O loop principal do jogo
void limparJogo(void); // Limpa a memoria (se necessario)
void mudarEstadoJogo(EstadoJogo novoEstado); // Troca de tela/fase
void processarEntrada(char entrada); // Decide o que fazer com o comando do jogador
void processarEntradaJogo(char entrada); // Processa comandos dentro das fases
void atualizarJogo(void); // Atualiza a logica do jogo (movimento de monstros, etc.)
void desenharJogo(void); // Mostra o jogo na tela
void reiniciarFase(void); // Volta pro inicio da fase atual
int jogoTerminou(void); // Verifica se o jogo acabou
void limparTela(void); // Limpa o console
void morteJogador(void); // O que acontece quando o jogador morre

void inicializarJogador(void); // Define a posicao inicial do jogador
int podeMoverPara(int linha, int coluna, int tamanhoMapa); // Verifica se o jogador pode ir para uma posicao
void moverJogador(int novaLinha, int novaColuna, int tamanhoMapa); // Move o jogador
void interagirComTile(int linha, int coluna, int tamanhoMapa); // Tenta interagir com algo proximo

void inicializarMapas(void); // Cria os mapas iniciais
void carregarMapaVila(void);
void carregarMapaFase1(void);
void carregarMapaFase2(void);
void carregarMapaFase3(void);
char obterTileMapa(int linha, int coluna, int tamanhoMapa); // Pega o caractere em uma posicao do mapa atual
char obterTileMapaBase(int linha, int coluna, int tamanhoMapa); // Pega o caractere do mapa original
void definirTileMapa(int linha, int coluna, char tile, int tamanhoMapa); // Coloca um caractere no mapa atual
void desenharMapa(int tamanhoMapa); // Desenha o mapa na tela
void abrirTodasPortas(int tamanhoMapa); // Abre as portas trancadas ('D')
void ativarBotao(int tamanhoMapa); // Ativa/desativa o botao
void encontrarDestinoTeletransporte(int linhaOrigem, int colunaOrigem, int* linhaDestino, int* colunaDestino, int tamanhoMapa); // Acha pra onde o teletransporte leva

void inicializarMonstros(void); // Prepara os monstros
void inicializarMonstro(int indice, int linha, int coluna, int tipo); // Cria um monstro especifico
void reiniciarMonstros(void); // Tira os monstros do mapa (quando reinicia a fase)
void atualizarMonstrosNivel1(void); // Movimenta os monstros tipo X
void atualizarMonstrosNivel2(void); // Movimenta os monstros tipo V
void verificarColisaoJogadorMonstro(int tamanhoMapa); // Verifica se o jogador trombou com um monstro

void desenharMenu(void);
void desenharCreditos(void);
void desenharVitoria(void);
void desenharDerrota(void);
void processarEntradaMenu(char entrada);
void processarEntradaCreditos(char entrada);

// --- Funcao Principal --- //
int main(void) {
    inicializarJogo(); // Prepara tudo
    executarJogo();    // Roda o jogo
    limparJogo();      // Limpa o que precisar no final
    return 0;
}

// --- Implementacao das Funcoes --- //

void inicializarJogo(void) {
    srand((unsigned int)time(NULL)); // Inicializa o gerador de numeros aleatorios
    inicializarMapas();
    inicializarJogador();
    inicializarMonstros();
    printf("Jogo inicializado. Preparado para a aventura?\n"); // Mensagem inicial
}

void executarJogo(void) {
    char entrada;
    
    limparTela(); // Limpa qualquer coisa que estivesse na tela antes
    
    while (jogoRodando) {
        desenharJogo(); // Mostra o estado atual
        
        printf("\nSua vez (W/A/S/D = Mover, I = Interagir, Q = Menu): ");
        scanf(" %c", &entrada); // Le o comando do jogador
        
        // Converte para minuscula pra facilitar
        if (entrada >= 'A' && entrada <= 'Z') {
            entrada = entrada - 'A' + 'a';
        }
        
        processarEntrada(entrada); // Processa o comando
        
        // So atualiza a logica se nao for pra sair ou ir pro menu
        if (estadoAtual != ESTADO_SAIR && estadoAtual != ESTADO_MENU && estadoAtual != ESTADO_CREDITOS) {
            atualizarJogo(); // Atualiza monstros, verifica colisoes, etc.
        }
        
        if (jogoTerminou()) { // Verifica se e pra parar o loop
            break;
        }
        
        // Nao limpa a tela imediatamente apos vitoria/derrota para o jogador ver a mensagem
        if (estadoAtual != ESTADO_VITORIA && estadoAtual != ESTADO_DERROTA) {
            limparTela();
        }
    }
}

void mudarEstadoJogo(EstadoJogo novoEstado) {
    printf("Mudando para o estado: %d\n", novoEstado); // Debug: informa a mudanca de estado
    
    // Se esta entrando numa fase (ou voltando pra vila), reseta algumas coisas
    if (novoEstado == ESTADO_FASE1 || novoEstado == ESTADO_FASE2 || 
        novoEstado == ESTADO_FASE3 || novoEstado == ESTADO_VILA) {
        temChave = 0; // Comeca sem chave
        contagemMortes = 0; // Zera o contador de mortes da fase
        reiniciarMonstros(); // Tira os monstros antigos
        botaoAtivo = 0; // Reseta o botao
    }
    
    estadoAtual = novoEstado;
    
    // Carrega o mapa correspondente ao novo estado
    switch (estadoAtual) {
        case ESTADO_VILA:
            carregarMapaVila();
            break;
        case ESTADO_FASE1:
            carregarMapaFase1();
            break;
        case ESTADO_FASE2:
            carregarMapaFase2();
            break;
        case ESTADO_FASE3:
            carregarMapaFase3();
            break;
        // Outros estados (Menu, Creditos, etc.) nao carregam mapa daqui
        default:
            break;
    }
    
    // Se o novo estado for uma fase, coloca o jogador na posicao inicial
    if (estadoAtual == ESTADO_VILA || estadoAtual == ESTADO_FASE1 || estadoAtual == ESTADO_FASE2 || estadoAtual == ESTADO_FASE3) {
        inicializarJogador(); // Garante que o jogador comece no lugar certo
        // Define o tile do jogador no mapa recem-carregado
        int tamanhoMapa = 0;
        if(estadoAtual == ESTADO_VILA) tamanhoMapa = TAMANHO_VILA;
        else if(estadoAtual == ESTADO_FASE1) tamanhoMapa = TAMANHO_FASE1;
        else if(estadoAtual == ESTADO_FASE2) tamanhoMapa = TAMANHO_FASE2;
        else if(estadoAtual == ESTADO_FASE3) tamanhoMapa = TAMANHO_FASE3;
        definirTileMapa(linhaJogador, colunaJogador, JOGADOR, tamanhoMapa);
    }
}

void processarEntrada(char entrada) {
    switch (estadoAtual) {
        case ESTADO_MENU:
            processarEntradaMenu(entrada);
            break;
        case ESTADO_CREDITOS:
            processarEntradaCreditos(entrada); // Qualquer tecla volta pro menu
            break;
        case ESTADO_SAIR:
            jogoRodando = 0; // Marca pra sair do loop principal
            break;
        case ESTADO_VILA:
        case ESTADO_FASE1:
        case ESTADO_FASE2:
        case ESTADO_FASE3:
            processarEntradaJogo(entrada); // Comandos dentro do jogo
            break;
        case ESTADO_VITORIA:
        case ESTADO_DERROTA:
            // Apos ver a tela de vitoria/derrota, qualquer tecla volta pro menu
            mudarEstadoJogo(ESTADO_MENU);
            break;
    }
}

void processarEntradaJogo(char entrada) {
    int novaLinha = linhaJogador;
    int novaColuna = colunaJogador;
    int tamanhoMapa = 0;
    
    // Descobre o tamanho do mapa atual
    switch (estadoAtual) {
        case ESTADO_VILA: tamanhoMapa = TAMANHO_VILA; break;
        case ESTADO_FASE1: tamanhoMapa = TAMANHO_FASE1; break;
        case ESTADO_FASE2: tamanhoMapa = TAMANHO_FASE2; break;
        case ESTADO_FASE3: tamanhoMapa = TAMANHO_FASE3; break;
        default: return; // Nao deveria acontecer, mas por seguranca...
    }
    
    // Calcula a nova posicao baseada na entrada
    switch (entrada) {
        case 'w': // Cima
            novaLinha = linhaJogador - 1;
            break;
        case 'a': // Esquerda
            novaColuna = colunaJogador - 1;
            break;
        case 's': // Baixo
            novaLinha = linhaJogador + 1;
            break;
        case 'd': // Direita
            novaColuna = colunaJogador + 1;
            break;
        case 'i': // Interagir
            interagirComTile(linhaJogador, colunaJogador, tamanhoMapa);
            return; // Interagir nao move, entao ja podemos sair
        case 'q': // Sair (Voltar pro Menu)
            mudarEstadoJogo(ESTADO_MENU);
            return; // Ja mudou de estado, nao precisa mover
        default: 
            printf("\nComando estranho... tente W, A, S, D, I ou Q.\n");
            return; // Comando invalido
    }
    
    // Verifica se a nova posicao e valida e se pode mover pra la
    if (novaLinha >= 0 && novaLinha < tamanhoMapa && 
        novaColuna >= 0 && novaColuna < tamanhoMapa && 
        podeMoverPara(novaLinha, novaColuna, tamanhoMapa)) {
        moverJogador(novaLinha, novaColuna, tamanhoMapa);
    } else if (entrada == 'w' || entrada == 'a' || entrada == 's' || entrada == 'd') {
        // Se tentou mover e nao pode (parede, etc.)
        printf("\nOps! Parece que tem algo bloqueando o caminho.\n");
    }
}

void atualizarJogo(void) {
    int tamanhoMapa = 0;
    
    // Logica especifica de cada fase
    switch (estadoAtual) {
        case ESTADO_VILA:
            // Nada de especial acontece na vila por enquanto
            break;
        case ESTADO_FASE1:
            // Fase 1 tambem e tranquila, sem monstros
            break;
        case ESTADO_FASE2:
            tamanhoMapa = TAMANHO_FASE2;
            atualizarMonstrosNivel1(); // Monstros X se movem
            verificarColisaoJogadorMonstro(tamanhoMapa); // Viu se trombou
            break;
        case ESTADO_FASE3:
            tamanhoMapa = TAMANHO_FASE3;
            atualizarMonstrosNivel1(); // Monstros X se movem
            atualizarMonstrosNivel2(); // Monstros V perseguem
            verificarColisaoJogadorMonstro(tamanhoMapa); // Viu se trombou
            break;
        default:
            return; // So atualiza se estiver numa fase
    }
    
    // Verifica se o jogador pisou em algo perigoso (espinho)
    if (estadoAtual == ESTADO_FASE2 || estadoAtual == ESTADO_FASE3) {
        if (obterTileMapa(linhaJogador, colunaJogador, tamanhoMapa) == ESPINHO) {
            printf("\nAi! Espinhos! Doi!\n");
            morteJogador();
            return; // Morreu, nao precisa checar mais nada nesta atualizacao
        }
    }
    
    // Verifica se o jogador chegou numa porta aberta (passou de fase)
    // Precisa checar tanto o mapa atual quanto o base, caso o jogador esteja sobre a porta
    char tileAtual = obterTileMapa(linhaJogador, colunaJogador, tamanhoMapa);
    char tileBase = obterTileMapaBase(linhaJogador, colunaJogador, tamanhoMapa);
    
    if ((estadoAtual == ESTADO_VILA || estadoAtual == ESTADO_FASE1 || 
         estadoAtual == ESTADO_FASE2 || estadoAtual == ESTADO_FASE3) && 
        (tileAtual == PORTA_ABERTA || tileBase == PORTA_ABERTA)) { // Se o jogador esta na saida
        
        printf("\nVoce encontrou a saida! Indo para o proximo desafio...\n");
        printf("Pressione Enter para continuar...");
        getchar(); // Pausa pra ler a mensagem
        getchar(); // Consome o Enter que ficou no buffer
        
        // Muda para a proxima fase ou tela de vitoria
        switch (estadoAtual) {
            case ESTADO_VILA: mudarEstadoJogo(ESTADO_FASE1); break;
            case ESTADO_FASE1: mudarEstadoJogo(ESTADO_FASE2); break;
            case ESTADO_FASE2: mudarEstadoJogo(ESTADO_FASE3); break;
            case ESTADO_FASE3: mudarEstadoJogo(ESTADO_VITORIA); break;
            default: break; // Seguranca
        }
    }
}

void reiniciarFase(void) {
    contagemMortes++; // Mais uma morte pra conta
    printf("Voce ja morreu %d vez(es) nesta area.\n", contagemMortes);
    
    if (contagemMortes >= 3) {
        printf("Suas chances acabaram... Fim de jogo.\n");
        mudarEstadoJogo(ESTADO_DERROTA);
        return;
    }
    
    printf("Tentando de novo...\n");
    
    // Recarrega o mapa do estado atual
    EstadoJogo estadoParaRecarregar = estadoAtual;
    botaoAtivo = 0; // Reseta o botao ao reiniciar
    temChave = 0; // Perde a chave ao morrer
    reiniciarMonstros(); // Tira os monstros
    
    switch (estadoParaRecarregar) {
        case ESTADO_VILA: // Nao deveria morrer na vila, mas...
            carregarMapaVila();
            break;
        case ESTADO_FASE1:
            carregarMapaFase1();
            break;
        case ESTADO_FASE2:
            carregarMapaFase2();
            break;
        case ESTADO_FASE3:
            carregarMapaFase3();
            break;
        default:
            // Se morrer em outro estado (?), volta pro menu por seguranca
            mudarEstadoJogo(ESTADO_MENU);
            return;
    }
    
    // Recoloca o jogador no inicio
    inicializarJogador();
    int tamanhoMapa = 0;
    if(estadoParaRecarregar == ESTADO_VILA) tamanhoMapa = TAMANHO_VILA;
    else if(estadoParaRecarregar == ESTADO_FASE1) tamanhoMapa = TAMANHO_FASE1;
    else if(estadoParaRecarregar == ESTADO_FASE2) tamanhoMapa = TAMANHO_FASE2;
    else if(estadoParaRecarregar == ESTADO_FASE3) tamanhoMapa = TAMANHO_FASE3;
    definirTileMapa(linhaJogador, colunaJogador, JOGADOR, tamanhoMapa);
    
    // Garante que o estado atual ainda e o da fase que foi reiniciada
    estadoAtual = estadoParaRecarregar; 
}

void morteJogador(void) {
    printf("\n----------------------------------------\n");
    printf("    Voce nao sobreviveu... Que pena!    \n");
    printf("\n----------------------------------------\n");
    printf("Pressione Enter para tentar novamente...");
    getchar(); // Pausa pra ler
    getchar(); // Consome o Enter
    reiniciarFase();
}

int jogoTerminou(void) {
    // O jogo termina se jogoRodando for 0 (escolheu sair no menu)
    return !jogoRodando;
}

void limparTela(void) {
    // Comando para limpar o console (funciona em Linux/Mac)
    // Em Windows, seria "cls"
    system("clear"); 
}

void limparJogo(void) {
    // Por enquanto, nao ha alocacao dinamica de memoria que precise ser liberada aqui.
    // Se adicionarmos coisas com malloc/calloc, teriamos que usar free() aqui.
    printf("\nJogo encerrado. Ate a proxima aventura!\n");
}

void inicializarJogador(void) {
    // Sempre comeca no canto superior esquerdo (tirando as paredes)
    linhaJogador = 1;
    colunaJogador = 1;
    // Nao coloca o JOGADOR no mapa aqui, isso e feito em mudarEstadoJogo ou reiniciarFase
}

int podeMoverPara(int linha, int coluna, int tamanhoMapa) {
    // Pega o que tem na celula de destino
    char tile = obterTileMapa(linha, coluna, tamanhoMapa);
    
    // Nao pode entrar em paredes ou portas fechadas
    if (tile == PAREDE || tile == PORTA_FECHADA) {
        return 0; // Nao pode
    }
    
    // Se nao for parede nem porta fechada, pode mover
    return 1; // Pode
}

void moverJogador(int novaLinha, int novaColuna, int tamanhoMapa) {
    // Guarda o que tinha na celula atual antes do jogador sair
    // Usa o mapa base para saber o que restaurar (nao apagar chaves, botoes, etc.)
    char tileAntigo = obterTileMapaBase(linhaJogador, colunaJogador, tamanhoMapa);
    
    // Limpa a posicao antiga do jogador, restaurando o que tinha la
    definirTileMapa(linhaJogador, colunaJogador, tileAntigo, tamanhoMapa);
    
    // Guarda a nova posicao
    linhaJogador = novaLinha;
    colunaJogador = novaColuna;
    
    // O que tem na nova celula?
    char tileNovo = obterTileMapa(linhaJogador, colunaJogador, tamanhoMapa);
    
    // Efeitos de pisar em certos tiles:
    if (tileNovo == CHAVE) {
        temChave = 1;
        printf("\nOpa! Uma chave! Deve servir para alguma porta por aqui.\n");
        // Remove a chave do mapa base tambem para nao reaparecer se sair e voltar
        mapaBase[linhaJogador][colunaJogador] = VAZIO;
    } else if (tileNovo == BOTAO) {
        printf("\n*Click* Voce pisou em um botao no chao.\n");
        // A logica de ativar o botao acontece na interacao ('i')
    } else if (tileNovo == ESPINHO) {
        // A verificacao de morte por espinho e feita em atualizarJogo
        printf("\nCuidado onde pisa! Espinhos!\n");
    } else if (tileNovo == TELETRANSPORTE) {
        printf("\nWhoosh! Voce entrou num portal de teletransporte!\n");
        int destLinha, destColuna;
        encontrarDestinoTeletransporte(linhaJogador, colunaJogador, &destLinha, &destColuna, tamanhoMapa);
        
        // Move o jogador para o destino do teletransporte
        linhaJogador = destLinha;
        colunaJogador = destColuna;
        printf("E saiu em (%d, %d)!\n", linhaJogador, colunaJogador);
        // Pega o tile do novo local apos teletransporte
        tileNovo = obterTileMapa(linhaJogador, colunaJogador, tamanhoMapa);
    } else if (tileNovo == MONSTRO_NIVEL1 || tileNovo == MONSTRO_NIVEL2) {
        // A colisao com monstro e verificada em atualizarJogo
        printf("\nUm monstro! Rapido!\n");
    }
    
    // Coloca o jogador na nova posicao no mapa atual
    definirTileMapa(linhaJogador, colunaJogador, JOGADOR, tamanhoMapa);
}

void interagirComTile(int linha, int coluna, int tamanhoMapa) {
    // Verifica se esta sobre um botao
    if (obterTileMapaBase(linha, coluna, tamanhoMapa) == BOTAO) {
        printf("\nVoce pisa com forca no botao!\n");
        ativarBotao(tamanhoMapa); // Ativa/desativa o efeito do botao
        printf("Algo parece ter mudado no ambiente...\n");
        return;
    }
    
    // Verifica se tem chave e esta perto de uma porta fechada
    if (temChave) {
        // Checa as 4 posicoes adjacentes
        int adjLinha[] = {linha-1, linha+1, linha, linha};
        int adjColuna[] = {coluna, coluna, coluna-1, coluna+1};
        
        for (int i = 0; i < 4; i++) {
            int l = adjLinha[i];
            int c = adjColuna[i];
            
            // Verifica se a posicao adjacente e valida e tem uma porta fechada
            if (l >= 0 && l < tamanhoMapa && c >= 0 && c < tamanhoMapa && 
                obterTileMapa(l, c, tamanhoMapa) == PORTA_FECHADA) {
                
                printf("\nVoce usa a chave... *Click!* A porta destrancou!\n");
                abrirTodasPortas(tamanhoMapa); // Abre a porta (e outras, se houver)
                temChave = 0; // Usou a chave
                return; // Interacao feita
            }
        }
    }
    
    // Verifica se esta perto de um personagem (NPC)
    int adjLinhaNPC[] = {linha-1, linha+1, linha, linha};
    int adjColunaNPC[] = {coluna, coluna, coluna-1, coluna+1};
    
    for (int i = 0; i < MAX_PERSONAGENS; i++) {
        if (personagens[i].ativo) { // Se o personagem existe na fase
            for (int j = 0; j < 4; j++) {
                int l = adjLinhaNPC[j];
                int c = adjColunaNPC[j];
                
                // Verifica se a posicao adjacente e valida e contem este personagem
                if (l >= 0 && l < tamanhoMapa && c >= 0 && c < tamanhoMapa && 
                    personagens[i].linha == l && personagens[i].coluna == c) {
                    
                    printf("\n[%c] diz: %s\n", PERSONAGEM, personagens[i].mensagem);
                    return; // Falou com o personagem
                }
            }
        }
    }
    
    // Se chegou ate aqui, nao ha nada interessante para interagir por perto
    printf("\nVoce olha ao redor, mas nao encontra nada para interagir aqui.\n");
}

void inicializarMapas(void) {
    // Limpa os personagens antigos antes de carregar um novo mapa
    for (int i = 0; i < MAX_PERSONAGENS; i++) {
        personagens[i].ativo = 0;
    }
    numTeletransportes = 0; // Zera teletransportes
    botaoAtivo = 0; // Zera botao
    
    // Comeca carregando a vila (o menu vai chamar mudarEstadoJogo depois se necessario)
    // carregarMapaVila(); // Removido daqui, e chamado em mudarEstadoJogo
}

// --- Funcoes de Carregar Mapas --- //
// Estas funcoes definem como cada mapa/fase e.
// Usam VAZIO, PAREDE, CHAVE, PORTA_FECHADA, PERSONAGEM, etc.
// Tambem preenchem o mapaBase com a estrutura fixa.

void carregarMapaVila(void) {
    int tam = TAMANHO_VILA;
    // Limpa o mapa atual e o base
    for (int i = 0; i < tam; i++) {
        for (int j = 0; j < tam; j++) {
            mapaVila[i][j] = VAZIO;
            mapaBase[i][j] = VAZIO;
        }
    }
    
    // Cria as paredes externas
    for (int i = 0; i < tam; i++) {
        mapaVila[0][i] = mapaBase[0][i] = PAREDE;
        mapaVila[tam-1][i] = mapaBase[tam-1][i] = PAREDE;
        mapaVila[i][0] = mapaBase[i][0] = PAREDE;
        mapaVila[i][tam-1] = mapaBase[i][tam-1] = PAREDE;
    }
    
    // Paredes internas (exemplo)
    mapaVila[3][3] = mapaBase[3][3] = PAREDE;
    mapaVila[3][4] = mapaBase[3][4] = PAREDE;
    mapaVila[3][5] = mapaBase[3][5] = PAREDE;
    mapaVila[3][6] = mapaBase[3][6] = PAREDE;
    mapaVila[7][2] = mapaBase[7][2] = PAREDE;
    mapaVila[7][3] = mapaBase[7][3] = PAREDE;
    mapaVila[7][4] = mapaBase[7][4] = PAREDE;
    mapaVila[6][4] = mapaBase[6][4] = PAREDE;
    
    // Itens e saida
    mapaVila[2][2] = mapaBase[2][2] = CHAVE;
    mapaVila[8][8] = mapaBase[8][8] = PORTA_FECHADA;
    
    // Personagens (NPCs)
    personagens[0].linha = 2; personagens[0].coluna = 7; personagens[0].ativo = 1;
    strcpy(personagens[0].mensagem, "Ola, aventureiro! Parece perdido? Use W,A,S,D pra andar e 'I' pra interagir com coisas.");
    
    personagens[1].linha = 5; personagens[1].coluna = 5; personagens[1].ativo = 1;
    strcpy(personagens[1].mensagem, "Ouvi dizer que tem uma chave por aqui... talvez abra aquela porta estranha?");
    
    personagens[2].linha = 7; personagens[2].coluna = 7; personagens[2].ativo = 1;
    strcpy(personagens[2].mensagem, "Cuidado la dentro! Dizem que as masmorras estao cheias de perigos...");
    
    // Coloca os personagens no mapa
    for (int i = 0; i < MAX_PERSONAGENS; i++) {
        if (personagens[i].ativo) {
            mapaVila[personagens[i].linha][personagens[i].coluna] = PERSONAGEM;
            mapaBase[personagens[i].linha][personagens[i].coluna] = PERSONAGEM; // Marca no base tambem
        }
    }
    
    // Posicao inicial do jogador (sera definida em inicializarJogador)
    // linhaJogador = 1; 
    // colunaJogador = 1;
    // mapaVila[linhaJogador][colunaJogador] = JOGADOR; // Feito em mudarEstadoJogo
}

void carregarMapaFase1(void) {
    int tam = TAMANHO_FASE1;
    for (int i = 0; i < tam; i++) {
        for (int j = 0; j < tam; j++) {
            mapaFase1[i][j] = VAZIO;
            mapaBase[i][j] = VAZIO;
        }
    }
    
    for (int i = 0; i < tam; i++) {
        mapaFase1[0][i] = mapaBase[0][i] = PAREDE;
        mapaFase1[tam-1][i] = mapaBase[tam-1][i] = PAREDE;
        mapaFase1[i][0] = mapaBase[i][0] = PAREDE;
        mapaFase1[i][tam-1] = mapaBase[i][tam-1] = PAREDE;
    }
    
    // Labirinto simples
    mapaFase1[2][2] = mapaBase[2][2] = PAREDE;
    mapaFase1[2][3] = mapaBase[2][3] = PAREDE;
    mapaFase1[2][4] = mapaBase[2][4] = PAREDE;
    mapaFase1[2][5] = mapaBase[2][5] = PAREDE;
    mapaFase1[4][1] = mapaBase[4][1] = PAREDE;
    mapaFase1[4][2] = mapaBase[4][2] = PAREDE;
    mapaFase1[4][3] = mapaBase[4][3] = PAREDE;
    mapaFase1[4][4] = mapaBase[4][4] = PAREDE;
    mapaFase1[4][6] = mapaBase[4][6] = PAREDE;
    mapaFase1[4][7] = mapaBase[4][7] = PAREDE;
    mapaFase1[4][8] = mapaBase[4][8] = PAREDE;
    mapaFase1[6][2] = mapaBase[6][2] = PAREDE;
    mapaFase1[6][3] = mapaBase[6][3] = PAREDE;
    mapaFase1[6][4] = mapaBase[6][4] = PAREDE;
    mapaFase1[6][5] = mapaBase[6][5] = PAREDE;
    mapaFase1[6][7] = mapaBase[6][7] = PAREDE;
    mapaFase1[6][8] = mapaBase[6][8] = PAREDE;
    mapaFase1[8][1] = mapaBase[8][1] = PAREDE;
    mapaFase1[8][2] = mapaBase[8][2] = PAREDE;
    mapaFase1[8][3] = mapaBase[8][3] = PAREDE;
    mapaFase1[8][4] = mapaBase[8][4] = PAREDE;
    mapaFase1[8][6] = mapaBase[8][6] = PAREDE;
    
    mapaFase1[7][7] = mapaBase[7][7] = CHAVE;
    mapaFase1[1][8] = mapaBase[1][8] = PORTA_FECHADA;
    
    // linhaJogador = 1; 
    // colunaJogador = 1;
    // mapaFase1[linhaJogador][colunaJogador] = JOGADOR; // Feito em mudarEstadoJogo
    botaoAtivo = 0; // Sem botao nesta fase
}

void carregarMapaFase2(void) {
    int tam = TAMANHO_FASE2;
    for (int i = 0; i < tam; i++) {
        for (int j = 0; j < tam; j++) {
            mapaFase2[i][j] = VAZIO;
            mapaBase[i][j] = VAZIO;
        }
    }
    
    for (int i = 0; i < tam; i++) {
        mapaFase2[0][i] = mapaBase[0][i] = PAREDE;
        mapaFase2[tam-1][i] = mapaBase[tam-1][i] = PAREDE;
        mapaFase2[i][0] = mapaBase[i][0] = PAREDE;
        mapaFase2[i][tam-1] = mapaBase[i][tam-1] = PAREDE;
    }
    
    // Paredes internas e obstaculos
    for (int i = 2; i < tam-2; i += 2) {
        for (int j = 2; j < tam-2; j += 4) {
            mapaFase2[i][j] = mapaBase[i][j] = PAREDE;
        }
    }
    for (int i = 5; i < tam-5; i += 3) {
        for (int j = 3; j < tam-3; j += 5) {
            mapaFase2[i][j] = mapaBase[i][j] = PAREDE;
        }
    }
    for (int i = 3; i < tam-3; i++) {
        if (i != 10) { // Deixa uma passagem
            mapaFase2[10][i] = mapaBase[10][i] = PAREDE;
        }
    }
    
    // Espinhos iniciais
    mapaFase2[4][4] = mapaBase[4][4] = ESPINHO;
    mapaFase2[4][15] = mapaBase[4][15] = ESPINHO;
    mapaFase2[15][4] = mapaBase[15][4] = ESPINHO;
    mapaFase2[15][15] = mapaBase[15][15] = ESPINHO;
    mapaFase2[8][12] = mapaBase[8][12] = ESPINHO;
    mapaFase2[12][8] = mapaBase[12][8] = ESPINHO;
    
    // Botao
    mapaFase2[18][10] = mapaBase[18][10] = BOTAO;
    
    // Espinhos que aparecem/somem com o botao
    if (botaoAtivo) { // Se o botao foi ativado (estado 1)
        // Neste caso, vamos fazer os espinhos SUMIREM quando ativa
        mapaBase[5][5] = VAZIO; mapaFase2[5][5] = VAZIO;
        mapaBase[5][14] = VAZIO; mapaFase2[5][14] = VAZIO;
        mapaBase[14][5] = VAZIO; mapaFase2[14][5] = VAZIO;
        mapaBase[14][14] = VAZIO; mapaFase2[14][14] = VAZIO;
        printf("DEBUG: Botao ATIVO, espinhos em (5,5), (5,14), (14,5), (14,14) DESATIVADOS.\n");
    } else { // Se o botao esta desativado (estado 0)
        // Espinhos aparecem
        mapaBase[5][5] = ESPINHO; mapaFase2[5][5] = ESPINHO;
        mapaBase[5][14] = ESPINHO; mapaFase2[5][14] = ESPINHO;
        mapaBase[14][5] = ESPINHO; mapaFase2[14][5] = ESPINHO;
        mapaBase[14][14] = ESPINHO; mapaFase2[14][14] = ESPINHO;
        printf("DEBUG: Botao INATIVO, espinhos em (5,5), (5,14), (14,5), (14,14) ATIVOS.\n");
    }
    
    // Chave e Porta
    mapaFase2[3][3] = mapaBase[3][3] = CHAVE;
    mapaFase2[18][18] = mapaBase[18][18] = PORTA_FECHADA;
    
    // Monstros (sao inicializados depois)
    reiniciarMonstros(); // Garante que nao tem monstros de antes
    inicializarMonstro(0, 7, 7, 0); // Monstro tipo X
    
    // linhaJogador = 1; 
    // colunaJogador = 1;
    // mapaFase2[linhaJogador][colunaJogador] = JOGADOR; // Feito em mudarEstadoJogo
}

void carregarMapaFase3(void) {
    int tam = TAMANHO_FASE3;
    numTeletransportes = 0; // Zera contador de teletransportes
    for (int i = 0; i < tam; i++) {
        for (int j = 0; j < tam; j++) {
            mapaFase3[i][j] = VAZIO;
            mapaBase[i][j] = VAZIO;
        }
    }
    
    for (int i = 0; i < tam; i++) {
        mapaFase3[0][i] = mapaBase[0][i] = PAREDE;
        mapaFase3[tam-1][i] = mapaBase[tam-1][i] = PAREDE;
        mapaFase3[i][0] = mapaBase[i][0] = PAREDE;
        mapaFase3[i][tam-1] = mapaBase[i][tam-1] = PAREDE;
    }
    
    // Estrutura mais complexa
    for (int i = 5; i < tam-5; i += 10) {
        for (int j = 1; j < tam-1; j++) {
            if (j != 20) { // Corredores horizontais com uma passagem vertical
                mapaFase3[i][j] = mapaBase[i][j] = PAREDE;
            }
        }
    }
    for (int i = 1; i < tam-1; i++) {
        for (int j = 10; j < tam-10; j += 15) {
            if (i % 10 != 5) { // Paredes verticais com passagens horizontais
                mapaFase3[i][j] = mapaBase[i][j] = PAREDE;
            }
        }
    }
    
    // Espinhos
    mapaFase3[10][15] = mapaBase[10][15] = ESPINHO;
    mapaFase3[15][30] = mapaBase[15][30] = ESPINHO;
    mapaFase3[25][10] = mapaBase[25][10] = ESPINHO;
    mapaFase3[30][25] = mapaBase[30][25] = ESPINHO;
    
    // Botao
    mapaFase3[15][15] = mapaBase[15][15] = BOTAO;
    
    // Teletransportes (par 1)
    mapaFase3[5][35] = mapaBase[5][35] = TELETRANSPORTE;
    posicoesTeletransporte[numTeletransportes][0] = 5;
    posicoesTeletransporte[numTeletransportes][1] = 35;
    numTeletransportes++;
    
    mapaFase3[35][5] = mapaBase[35][5] = TELETRANSPORTE;
    posicoesTeletransporte[numTeletransportes][0] = 35;
    posicoesTeletransporte[numTeletransportes][1] = 5;
    numTeletransportes++;
    // Adicione mais pares se precisar, lembrando de atualizar MAX_TELETRANSPORTES
    
    // Efeito do botao: remove/coloca um espinho central
    if (botaoAtivo) { // Botao ativo: remove o espinho
        mapaFase3[20][20] = VAZIO;
        mapaBase[20][20] = VAZIO;
        printf("DEBUG: Botao ATIVO, espinho em (20,20) DESATIVADO.\n");
    } else { // Botao inativo: coloca o espinho
        mapaFase3[20][20] = ESPINHO;
        mapaBase[20][20] = ESPINHO;
        printf("DEBUG: Botao INATIVO, espinho em (20,20) ATIVO.\n");
    }
    
    // Chave e Porta
    mapaFase3[20][35] = mapaBase[20][35] = CHAVE;
    mapaFase3[5][5] = mapaBase[5][5] = PORTA_FECHADA;
    
    // Monstros
    reiniciarMonstros();
    inicializarMonstro(0, 10, 10, 0); // Monstro X
    inicializarMonstro(1, 30, 30, 1); // Monstro V (perseguidor)
    
    // linhaJogador = 1; 
    // colunaJogador = 1;
    // mapaFase3[linhaJogador][colunaJogador] = JOGADOR; // Feito em mudarEstadoJogo
}

// --- Funcoes de Manipulacao do Mapa --- //

char obterTileMapa(int linha, int coluna, int tamanhoMapa) {
    // Verifica limites para nao ler fora do array
    if (linha < 0 || coluna < 0 || linha >= tamanhoMapa || coluna >= tamanhoMapa) {
        return PAREDE; // Trata fora do mapa como parede
    }
    
    // Retorna o caractere do mapa correspondente ao estado atual
    switch (estadoAtual) {
        case ESTADO_VILA: return mapaVila[linha][coluna];
        case ESTADO_FASE1: return mapaFase1[linha][coluna];
        case ESTADO_FASE2: return mapaFase2[linha][coluna];
        case ESTADO_FASE3: return mapaFase3[linha][coluna];
        default: return VAZIO; // Estado invalido? Retorna vazio.
    }
}

char obterTileMapaBase(int linha, int coluna, int tamanhoMapa) {
    // Verifica limites
    if (linha < 0 || coluna < 0 || linha >= tamanhoMapa || coluna >= tamanhoMapa) {
        return PAREDE;
    }
    // Sempre retorna do mapaBase, que nao muda com jogador/monstros
    return mapaBase[linha][coluna];
}

void definirTileMapa(int linha, int coluna, char tile, int tamanhoMapa) {
    // Verifica limites
    if (linha < 0 || coluna < 0 || linha >= tamanhoMapa || coluna >= tamanhoMapa) {
        return; // Nao faz nada se for fora do mapa
    }
    
    // Define o caractere no mapa do estado atual
    switch (estadoAtual) {
        case ESTADO_VILA: mapaVila[linha][coluna] = tile; break;
        case ESTADO_FASE1: mapaFase1[linha][coluna] = tile; break;
        case ESTADO_FASE2: mapaFase2[linha][coluna] = tile; break;
        case ESTADO_FASE3: mapaFase3[linha][coluna] = tile; break;
        default: break; // Nao faz nada em outros estados
    }
}

void desenharMapa(int tamanhoMapa) {
    printf("\nMapa Atual:\n");
    for (int i = 0; i < tamanhoMapa; i++) {
        printf("  "); // Um pequeno recuo pra ficar mais bonito
        for (int j = 0; j < tamanhoMapa; j++) {
            printf("%c ", obterTileMapa(i, j, tamanhoMapa)); // Imprime o caractere e um espaco
        }
        printf("\n"); // Nova linha no final de cada linha do mapa
    }
    // Mostra informacoes uteis
    printf("Posicao: (%d, %d) | Chave: %s | Vidas: %d/3\n", 
           linhaJogador, colunaJogador, 
           temChave ? "Sim" : "Nao", 
           3 - contagemMortes);
}

void abrirTodasPortas(int tamanhoMapa) {
    for (int i = 0; i < tamanhoMapa; i++) {
        for (int j = 0; j < tamanhoMapa; j++) {
            // Se encontrar uma porta fechada no mapa atual...
            if (obterTileMapa(i, j, tamanhoMapa) == PORTA_FECHADA) {
                definirTileMapa(i, j, PORTA_ABERTA, tamanhoMapa); // Abre no mapa atual
                mapaBase[i][j] = PORTA_ABERTA; // Marca como aberta no base tambem
            }
        }
    }
}

void ativarBotao(int tamanhoMapa) {
    botaoAtivo = !botaoAtivo; // Inverte o estado do botao (0 vira 1, 1 vira 0)
    printf("DEBUG: Botao agora esta %s\n", botaoAtivo ? "ATIVO" : "INATIVO");
    
    // Recarrega o mapa da fase atual para aplicar o efeito do botao
    // Guarda a posicao atual do jogador pra recoloca-lo depois
    int lJog = linhaJogador;
    int cJog = colunaJogador;
    
    switch (estadoAtual) {
        case ESTADO_FASE2:
            carregarMapaFase2(); 
            break;
        case ESTADO_FASE3:
            carregarMapaFase3();
            break;
        default:
            // Botao nao faz nada em outras fases/estados
            return;
    }
    
    // Recoloca o jogador na posicao que estava
    linhaJogador = lJog;
    colunaJogador = cJog;
    definirTileMapa(linhaJogador, colunaJogador, JOGADOR, tamanhoMapa);
    
    // Reinicializa os monstros da fase conforme o novo layout
    // (carregarMapaFaseX ja faz isso)
}

void encontrarDestinoTeletransporte(int linhaOrigem, int colunaOrigem, int* linhaDestino, int* colunaDestino, int tamanhoMapa) {
    int indiceOrigem = -1;
    // Procura qual teletransporte e o de origem
    for (int i = 0; i < numTeletransportes; i++) {
        if (posicoesTeletransporte[i][0] == linhaOrigem && posicoesTeletransporte[i][1] == colunaOrigem) {
            indiceOrigem = i;
            break;
        }
    }
    
    // Se nao achou (nao deveria acontecer se pisou num '>'), retorna a propria posicao
    if (indiceOrigem == -1) {
        *linhaDestino = linhaOrigem;
        *colunaDestino = colunaOrigem;
        printf("DEBUG: Erro! Pisei em teletransporte nao registrado?\n");
        return;
    }
    
    // Acha o indice do destino: se origem e par, destino e impar seguinte; se origem e impar, destino e par anterior.
    int indiceDestino = (indiceOrigem % 2 == 0) ? indiceOrigem + 1 : indiceOrigem - 1;
    
    // Verifica se o indice de destino e valido
    if (indiceDestino < 0 || indiceDestino >= numTeletransportes) {
        *linhaDestino = linhaOrigem; // Falha: volta pra origem
        *colunaDestino = colunaOrigem;
        printf("DEBUG: Erro! Teletransporte sem par? Indice Destino: %d\n", indiceDestino);
        return;
    }
    
    // Define as coordenadas de destino
    *linhaDestino = posicoesTeletransporte[indiceDestino][0];
    *colunaDestino = posicoesTeletransporte[indiceDestino][1];
}

// --- Funcoes dos Monstros --- //

void inicializarMonstros(void) {
    // Zera todos os monstros no inicio de cada fase
    for (int i = 0; i < MAX_MONSTROS; i++) {
        monstros[i].ativo = 0;
    }
}

void inicializarMonstro(int indice, int linha, int coluna, int tipo) {
    // Verifica se o indice e valido
    if (indice < 0 || indice >= MAX_MONSTROS) {
        printf("DEBUG: Tentativa de criar monstro com indice invalido: %d\n", indice);
        return;
    }
    
    monstros[indice].linha = linha;
    monstros[indice].coluna = coluna;
    monstros[indice].ativo = 1; // Monstro esta vivo e no jogo
    monstros[indice].tipo = tipo; // 0 = Nivel 1 (X), 1 = Nivel 2 (V)
    
    // Coloca o monstro no mapa atual
    int tamanhoMapa = 0;
    char simboloMonstro = (tipo == 0) ? MONSTRO_NIVEL1 : MONSTRO_NIVEL2;
    
    switch(estadoAtual) {
        case ESTADO_FASE2: tamanhoMapa = TAMANHO_FASE2; definirTileMapa(linha, coluna, simboloMonstro, tamanhoMapa); break;
        case ESTADO_FASE3: tamanhoMapa = TAMANHO_FASE3; definirTileMapa(linha, coluna, simboloMonstro, tamanhoMapa); break;
        // Monstros so existem nas fases 2 e 3 por enquanto
        default: break;
    }
}

void reiniciarMonstros(void) {
    // Tira os monstros do mapa e marca como inativos
    int tamanhoMapa = 0;
    switch(estadoAtual) {
        case ESTADO_FASE2: tamanhoMapa = TAMANHO_FASE2; break;
        case ESTADO_FASE3: tamanhoMapa = TAMANHO_FASE3; break;
        default: return; // Sem monstros em outros estados
    }
    
    for (int i = 0; i < MAX_MONSTROS; i++) {
        if (monstros[i].ativo) {
            // Se o tile atual e o monstro, limpa (restaura do base)
            if (obterTileMapa(monstros[i].linha, monstros[i].coluna, tamanhoMapa) == MONSTRO_NIVEL1 ||
                obterTileMapa(monstros[i].linha, monstros[i].coluna, tamanhoMapa) == MONSTRO_NIVEL2) {
                definirTileMapa(monstros[i].linha, monstros[i].coluna, obterTileMapaBase(monstros[i].linha, monstros[i].coluna, tamanhoMapa), tamanhoMapa);
            }
            monstros[i].ativo = 0; // Marca como inativo
        }
    }
}

void atualizarMonstrosNivel1(void) {
    int tamanhoMapa = (estadoAtual == ESTADO_FASE2) ? TAMANHO_FASE2 : TAMANHO_FASE3;
    
    for (int i = 0; i < MAX_MONSTROS; i++) {
        // So mexe com monstros ativos do tipo 0 (X)
        if (monstros[i].ativo && monstros[i].tipo == 0) {
            // Limpa a posicao antiga do monstro (restaura o que tinha no mapa base)
            char tileBaseAntigo = obterTileMapaBase(monstros[i].linha, monstros[i].coluna, tamanhoMapa);
            definirTileMapa(monstros[i].linha, monstros[i].coluna, tileBaseAntigo, tamanhoMapa);
            
            // Tenta mover aleatoriamente
            int direcao = rand() % 4; // 0: Cima, 1: Baixo, 2: Esquerda, 3: Direita
            int novaLinha = monstros[i].linha;
            int novaColuna = monstros[i].coluna;
            
            switch (direcao) {
                case 0: novaLinha--; break;
                case 1: novaLinha++; break;
                case 2: novaColuna--; break;
                case 3: novaColuna++; break;
            }
            
            // Verifica se pode mover para a nova posicao
            char novoTile = obterTileMapa(novaLinha, novaColuna, tamanhoMapa);
            if (novoTile != PAREDE && novoTile != PORTA_FECHADA && novoTile != PORTA_ABERTA &&
                novoTile != CHAVE && novoTile != BOTAO && novoTile != TELETRANSPORTE &&
                novoTile != MONSTRO_NIVEL1 && novoTile != MONSTRO_NIVEL2 && novoTile != PERSONAGEM &&
                novoTile != JOGADOR) { // Nao entra em cima de outros monstros, itens ou jogador (a colisao e separada)
                
                // Atualiza a posicao do monstro
                monstros[i].linha = novaLinha;
                monstros[i].coluna = novaColuna;
            }
            // Se nao pode mover, ele fica parado nesta rodada
            
            // Coloca o monstro na (nova ou antiga) posicao no mapa atual
            definirTileMapa(monstros[i].linha, monstros[i].coluna, MONSTRO_NIVEL1, tamanhoMapa);
        }
    }
}

void atualizarMonstrosNivel2(void) {
    int tamanhoMapa = TAMANHO_FASE3; // Monstro tipo V so na fase 3 por enquanto
    if (estadoAtual != ESTADO_FASE3) return;
    
    for (int i = 0; i < MAX_MONSTROS; i++) {
        // So mexe com monstros ativos do tipo 1 (V)
        if (monstros[i].ativo && monstros[i].tipo == 1) {
            // Limpa a posicao antiga
            char tileBaseAntigo = obterTileMapaBase(monstros[i].linha, monstros[i].coluna, tamanhoMapa);
            definirTileMapa(monstros[i].linha, monstros[i].coluna, tileBaseAntigo, tamanhoMapa);
            
            // Tenta mover em direcao ao jogador
            int difLinha = linhaJogador - monstros[i].linha;
            int difColuna = colunaJogador - monstros[i].coluna;
            
            int novaLinha = monstros[i].linha;
            int novaColuna = monstros[i].coluna;
            
            // Move na direcao predominante (vertical ou horizontal)
            if (abs(difLinha) > abs(difColuna)) { // Move na vertical primeiro
                novaLinha += (difLinha > 0) ? 1 : -1;
            } else if (abs(difColuna) > abs(difLinha)) { // Move na horizontal primeiro
                novaColuna += (difColuna > 0) ? 1 : -1;
            } else if (difLinha != 0) { // Empate, tenta mover na vertical (se nao for 0)
                 novaLinha += (difLinha > 0) ? 1 : -1;
            } else if (difColuna != 0) { // Empate, tenta mover na horizontal (se nao for 0)
                 novaColuna += (difColuna > 0) ? 1 : -1;
            } 
            // Se difLinha e difColuna sao 0, o monstro esta no mesmo lugar que o jogador (colisao tratada depois)

            // Verifica se a posicao calculada e valida
            char novoTile = obterTileMapa(novaLinha, novaColuna, tamanhoMapa);
            int podeMoverDireto = (novoTile != PAREDE && novoTile != PORTA_FECHADA && novoTile != PORTA_ABERTA &&
                                   novoTile != CHAVE && novoTile != BOTAO && novoTile != TELETRANSPORTE &&
                                   novoTile != MONSTRO_NIVEL1 && novoTile != MONSTRO_NIVEL2 && novoTile != PERSONAGEM &&
                                   novoTile != JOGADOR);

            if (podeMoverDireto) {
                // Conseguiu mover na direcao do jogador
                monstros[i].linha = novaLinha;
                monstros[i].coluna = novaColuna;
            } else {
                // Nao conseguiu mover direto (parede, outro monstro, etc.)
                // Tenta mover na outra direcao como alternativa
                novaLinha = monstros[i].linha; // Reseta pra posicao original
                novaColuna = monstros[i].coluna;

                if (abs(difLinha) > abs(difColuna)) { // Tentou vertical, falhou, tenta horizontal
                    novaColuna += (difColuna > 0) ? 1 : -1;
                } else { // Tentou horizontal (ou empate), falhou, tenta vertical
                    novaLinha += (difLinha > 0) ? 1 : -1;
                }
                
                // Verifica de novo se a alternativa e valida
                novoTile = obterTileMapa(novaLinha, novaColuna, tamanhoMapa);
                int podeMoverAlternativa = (novoTile != PAREDE && novoTile != PORTA_FECHADA && novoTile != PORTA_ABERTA &&
                                           novoTile != CHAVE && novoTile != BOTAO && novoTile != TELETRANSPORTE &&
                                           novoTile != MONSTRO_NIVEL1 && novoTile != MONSTRO_NIVEL2 && novoTile != PERSONAGEM &&
                                           novoTile != JOGADOR);
                
                if (podeMoverAlternativa) {
                    monstros[i].linha = novaLinha;
                    monstros[i].coluna = novaColuna;
                } else {
                    // Se ambas as direcoes falharam, fica parado
                    // (Poderia tentar movimento aleatorio aqui, mas vamos deixar simples)
                }
            }
            
            // Coloca o monstro na (nova ou antiga) posicao no mapa atual
            definirTileMapa(monstros[i].linha, monstros[i].coluna, MONSTRO_NIVEL2, tamanhoMapa);
        }
    }
}

void verificarColisaoJogadorMonstro(int tamanhoMapa) {
    for (int i = 0; i < MAX_MONSTROS; i++) {
        // Se um monstro ativo esta na mesma posicao do jogador...
        if (monstros[i].ativo && 
            monstros[i].linha == linhaJogador && 
            monstros[i].coluna == colunaJogador) {
            
            printf("\nArgh! Voce foi pego por um %s!\n", (monstros[i].tipo == 0) ? "Monstro (X)" : "Monstro (V)");
            morteJogador(); // Ja era
            return; // Sai da funcao, pois o jogador morreu
        }
    }
}

// --- Funcoes de Desenhar Telas --- //

void desenharJogo(void) {
    // Limpa a tela ANTES de desenhar o novo estado
    // Exceto para vitoria/derrota, onde esperamos o input antes de limpar
    if (estadoAtual != ESTADO_VITORIA && estadoAtual != ESTADO_DERROTA) {
        limparTela();
    }
    
    switch (estadoAtual) {
        case ESTADO_MENU:
            desenharMenu();
            break;
        case ESTADO_CREDITOS:
            desenharCreditos();
            break;
        case ESTADO_SAIR:
            // A mensagem de sair e mostrada em limparJogo()
            break;
        case ESTADO_VILA:
            printf("\n--- Explorando a Vila ---\n");
            desenharMapa(TAMANHO_VILA);
            // printf("\nVidas restantes: Infinitas na Vila\n"); // Removido, ja mostra no desenharMapa
            break;
        case ESTADO_FASE1:
            printf("\n--- Masmorra - Nivel 1 ---\n");
            desenharMapa(TAMANHO_FASE1);
            // printf("\nVidas restantes: %d\n", 3 - contagemMortes); // Removido
            break;
        case ESTADO_FASE2:
            printf("\n--- Masmorra - Nivel 2 ---\n");
            desenharMapa(TAMANHO_FASE2);
            // printf("\nVidas restantes: %d\n", 3 - contagemMortes); // Removido
            break;
        case ESTADO_FASE3:
            printf("\n--- Masmorra - Nivel 3 (Final) ---\n");
            desenharMapa(TAMANHO_FASE3);
            // printf("\nVidas restantes: %d\n", 3 - contagemMortes); // Removido
            break;
        case ESTADO_VITORIA:
            desenharVitoria();
            break;
        case ESTADO_DERROTA:
            desenharDerrota();
            break;
    }
}

void desenharMenu(void) {
    printf("\n**************************************************\n");
    printf("* *\n");
    printf("* BEMVINDO AO DUNGEON CRAWLER             *\n"); // Hyphen removed from BEM-VINDO
    printf("* *\n");
    printf("**************************************************\n");
    printf("\n                  O que voce deseja fazer?\n");
    printf("\n                  [1] Iniciar Aventura\n");
    printf("                  [2] Ver Creditos\n");
    printf("                  [3] Desistir (Sair)\n");
    printf("\n                  Digite o numero da opcao: ");
}

void desenharCreditos(void) {
    printf("\n=================== CREDITOS ===================\n");
    printf("\n    Este humilde jogo foi criado por:\n");
    printf("        > Athos Ramos\n");
    printf("        > Bruno Henrique\n");
    printf("        > Josiel Teixeira\n");
    printf("\n    Inspirado em classicos jogos de masmorra.\n");
    printf("\n==================================================\n");
    printf("\nPressione Enter para voltar ao Menu Principal...");
}

void desenharVitoria(void) {
    printf("\n##################################################\n");
    printf("#                                                #\n");
    printf("#                    VITORIA!!!                  #\n");
    printf("#                                                #\n");
    printf("##################################################\n");
    printf("\n    Parabens, grande aventureiro(a)!\n");
    printf("    Voce desbravou as profundezas da masmorra\n");
    printf("    e superou todos os desafios! Incrivel!\n");
    printf("\n    O tesouro (imaginario) e todo seu!\n");
    printf("\n##################################################\n");
    printf("\nPressione Enter para retornar ao Menu Principal...");
}

void desenharDerrota(void) {
    printf("\n++++++++++++++++++++++++++++++++++++++++++++++++++\n");
    printf("+                                                +\n");
    printf("+                   FIM DE JOGO                  +\n");
    printf("+                                                +\n");
    printf("++++++++++++++++++++++++++++++++++++++++++++++++++\n");
    printf("\n    Ah, que pena... A masmorra levou a melhor.\n");
    printf("    Voce nao conseguiu completar sua missao desta vez.\n");
    printf("\n    Mas nao desanime! Quem sabe na proxima?\n");
    printf("\n++++++++++++++++++++++++++++++++++++++++++++++++++\n");
    printf("\nPressione Enter para retornar ao Menu Principal...");
}

// --- Funcoes de Processar Entradas Especificas --- //

void processarEntradaMenu(char entrada) {
    switch (entrada) {
        case '1':
            printf("\nIniciando a aventura! Prepare-se...\n");
            mudarEstadoJogo(ESTADO_VILA); // Comeca na vila
            break;
        case '2':
            mudarEstadoJogo(ESTADO_CREDITOS);
            break;
        case '3':
            mudarEstadoJogo(ESTADO_SAIR);
            jogoRodando = 0; // Marca para o loop principal terminar
            break;
        default:
            printf("\nOpcao invalida. Por favor, escolha 1, 2 ou 3.\n");
            printf("Pressione Enter para continuar...");
            getchar(); // Pausa pra ler
            getchar(); // Consome o Enter
            break;
    }
}

void processarEntradaCreditos(char entrada) {
    // Qualquer tecla pressionada na tela de creditos volta para o menu
    mudarEstadoJogo(ESTADO_MENU);
}
void processarEntradaCreditos(char entrada) {
    mudarEstadoJogo(ESTADO_MENU);
}
