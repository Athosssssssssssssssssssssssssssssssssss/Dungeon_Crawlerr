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
#define MONSTRO_NIVEL1 'X' 
#define MONSTRO_NIVEL2 'V' 
#define VAZIO ' '
#define TAMANHO_VILA 10
#define TAMANHO_FASE1 10
#define TAMANHO_FASE2 20
#define TAMANHO_FASE3 40
#define TAMANHO_MAXIMO_MAPA 40
#define MAX_MONSTROS 10
#define MAX_TELETRANSPORTES 4
#define MAX_PERSONAGENS 5
typedef struct {
    int linha;
    int coluna;
    int ativo; 
    char mensagem[100]; 
} Personagem;
typedef struct {
    int linha;
    int coluna;
    int ativo; 
    int tipo;  
} Monstro;
EstadoJogo estadoAtual = ESTADO_MENU; 
int jogoRodando = 1; 
int contagemMortes = 0; 
int temChave = 0; 
int linhaJogador, colunaJogador; 
char mapaVila[TAMANHO_VILA][TAMANHO_VILA];
char mapaFase1[TAMANHO_FASE1][TAMANHO_FASE1];
char mapaFase2[TAMANHO_FASE2][TAMANHO_FASE2];
char mapaFase3[TAMANHO_FASE3][TAMANHO_FASE3];
char mapaBase[TAMANHO_MAXIMO_MAPA][TAMANHO_MAXIMO_MAPA]; 
Personagem personagens[MAX_PERSONAGENS]; 
Monstro monstros[MAX_MONSTROS]; 
int posicoesTeletransporte[MAX_TELETRANSPORTES][2]; 
int numTeletransportes = 0; 
int botaoAtivo = 0; 
void inicializarJogo(void); 
void executarJogo(void); 
void limparJogo(void); 
void mudarEstadoJogo(EstadoJogo novoEstado); 
void processarEntrada(char entrada); 
void processarEntradaJogo(char entrada); 
void atualizarJogo(void); 
void desenharJogo(void); 
void reiniciarFase(void); 
int jogoTerminou(void); 
void limparTela(void); 
void morteJogador(void); 
void inicializarJogador(void); 
int podeMoverPara(int linha, int coluna, int tamanhoMapa); 
void moverJogador(int novaLinha, int novaColuna, int tamanhoMapa); 
void interagirComTile(int linha, int coluna, int tamanhoMapa); 
void inicializarMapas(void); 
void carregarMapaVila(void);
void carregarMapaFase1(void);
void carregarMapaFase2(void);
void carregarMapaFase3(void);
char obterTileMapa(int linha, int coluna, int tamanhoMapa); 
char obterTileMapaBase(int linha, int coluna, int tamanhoMapa); 
void definirTileMapa(int linha, int coluna, char tile, int tamanhoMapa); 
void desenharMapa(int tamanhoMapa); 
void abrirTodasPortas(int tamanhoMapa); 
void ativarBotao(int tamanhoMapa); 
void encontrarDestinoTeletransporte(int linhaOrigem, int colunaOrigem, int* linhaDestino, int* colunaDestino, int tamanhoMapa); 
void inicializarMonstros(void); 
void inicializarMonstro(int indice, int linha, int coluna, int tipo); 
void reiniciarMonstros(void); 
void atualizarMonstrosNivel1(void); 
void atualizarMonstrosNivel2(void); 
void verificarColisaoJogadorMonstro(int tamanhoMapa); 
void desenharMenu(void);
void desenharCreditos(void);
void desenharVitoria(void);
void desenharDerrota(void);
void processarEntradaMenu(char entrada);
void processarEntradaCreditos(char entrada);
int main(void) {
    inicializarJogo(); 
    executarJogo();    
    limparJogo();      
    return 0;
}
void inicializarJogo(void) {
    srand((unsigned int)time(NULL)); 
    inicializarMapas();
    inicializarJogador();
    inicializarMonstros();
    printf("Jogo inicializado. Preparado para a aventura?\n"); 
}
void executarJogo(void) {
    char entrada;
    limparTela(); 
    while (jogoRodando) {
        desenharJogo(); 
        printf("\nSua vez (W/A/S/D = Mover, I = Interagir, Q = Menu): ");
        scanf(" %c", &entrada); 
        if (entrada >= 'A' && entrada <= 'Z') {
            entrada = entrada - 'A' + 'a';
        }
        processarEntrada(entrada); 
        if (estadoAtual != ESTADO_SAIR && estadoAtual != ESTADO_MENU && estadoAtual != ESTADO_CREDITOS) {
             atualizarJogo(); 
        }
        if (jogoTerminou()) { 
            break;
        }
        if (estadoAtual != ESTADO_VITORIA && estadoAtual != ESTADO_DERROTA) {
             limparTela();
        }
    }
}
void mudarEstadoJogo(EstadoJogo novoEstado) {
    printf("Mudando para o estado: %d\n", novoEstado); 
    if (novoEstado == ESTADO_FASE1 || novoEstado == ESTADO_FASE2 || 
        novoEstado == ESTADO_FASE3 || novoEstado == ESTADO_VILA) {
        temChave = 0; 
        contagemMortes = 0; 
        reiniciarMonstros(); 
        botaoAtivo = 0; 
    }
    estadoAtual = novoEstado;
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
        default:
            break;
    }
    if (estadoAtual == ESTADO_VILA || estadoAtual == ESTADO_FASE1 || estadoAtual == ESTADO_FASE2 || estadoAtual == ESTADO_FASE3) {
        inicializarJogador(); 
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
            processarEntradaCreditos(entrada); 
            break;
        case ESTADO_SAIR:
            jogoRodando = 0; 
            break;
        case ESTADO_VILA:
        case ESTADO_FASE1:
        case ESTADO_FASE2:
        case ESTADO_FASE3:
            processarEntradaJogo(entrada); 
            break;
        case ESTADO_VITORIA:
        case ESTADO_DERROTA:
            mudarEstadoJogo(ESTADO_MENU);
            break;
    }
}
void processarEntradaJogo(char entrada) {
    int novaLinha = linhaJogador;
    int novaColuna = colunaJogador;
    int tamanhoMapa = 0;
    switch (estadoAtual) {
        case ESTADO_VILA: tamanhoMapa = TAMANHO_VILA; break;
        case ESTADO_FASE1: tamanhoMapa = TAMANHO_FASE1; break;
        case ESTADO_FASE2: tamanhoMapa = TAMANHO_FASE2; break;
        case ESTADO_FASE3: tamanhoMapa = TAMANHO_FASE3; break;
        default: return; 
    }
    switch (entrada) {
        case 'w': 
            novaLinha = linhaJogador - 1;
            break;
        case 'a': 
            novaColuna = colunaJogador - 1;
            break;
        case 's': 
            novaLinha = linhaJogador + 1;
            break;
        case 'd': 
            novaColuna = colunaJogador + 1;
            break;
        case 'i': 
            interagirComTile(linhaJogador, colunaJogador, tamanhoMapa);
            return; 
        case 'q': 
            mudarEstadoJogo(ESTADO_MENU);
            return; 
        default: 
            printf("\nComando estranho... tente W, A, S, D, I ou Q.\n");
            return; 
    }
    if (novaLinha >= 0 && novaLinha < tamanhoMapa && 
        novaColuna >= 0 && novaColuna < tamanhoMapa && 
        podeMoverPara(novaLinha, novaColuna, tamanhoMapa)) {
        moverJogador(novaLinha, novaColuna, tamanhoMapa);
    } else if (entrada == 'w' || entrada == 'a' || entrada == 's' || entrada == 'd') {
        printf("\nOps! Parece que tem algo bloqueando o caminho.\n");
    }
}
void atualizarJogo(void) {
    int tamanhoMapa = 0;
    switch (estadoAtual) {
        case ESTADO_VILA:
            break;
        case ESTADO_FASE1:
            break;
        case ESTADO_FASE2:
            tamanhoMapa = TAMANHO_FASE2;
            atualizarMonstrosNivel1(); 
            verificarColisaoJogadorMonstro(tamanhoMapa); 
            break;
        case ESTADO_FASE3:
            tamanhoMapa = TAMANHO_FASE3;
            atualizarMonstrosNivel1(); 
            atualizarMonstrosNivel2(); 
            verificarColisaoJogadorMonstro(tamanhoMapa); 
            break;
        default:
            return; 
    }
    if (estadoAtual == ESTADO_FASE2 || estadoAtual == ESTADO_FASE3) {
        if (obterTileMapa(linhaJogador, colunaJogador, tamanhoMapa) == ESPINHO) {
            printf("\nAiii! Espinhos! DÃ³i!\n");
            morteJogador();
            return; 
        }
    }
    char tileAtual = obterTileMapa(linhaJogador, colunaJogador, tamanhoMapa);
    char tileBase = obterTileMapaBase(linhaJogador, colunaJogador, tamanhoMapa);
    if ((estadoAtual == ESTADO_VILA || estadoAtual == ESTADO_FASE1 || 
         estadoAtual == ESTADO_FASE2 || estadoAtual == ESTADO_FASE3) && 
        (tileAtual == PORTA_ABERTA || tileBase == PORTA_ABERTA)) { 
        printf("\nVocÃª encontrou a saÃ­da! Indo para o prÃ³ximo desafio...\n");
        printf("Pressione Enter para continuar...");
        getchar(); 
        getchar(); 
        switch (estadoAtual) {
            case ESTADO_VILA: mudarEstadoJogo(ESTADO_FASE1); break;
            case ESTADO_FASE1: mudarEstadoJogo(ESTADO_FASE2); break;
            case ESTADO_FASE2: mudarEstadoJogo(ESTADO_FASE3); break;
            case ESTADO_FASE3: mudarEstadoJogo(ESTADO_VITORIA); break;
            default: break; 
        }
    }
}
void reiniciarFase(void) {
    contagemMortes++; 
    printf("VocÃª jÃ¡ morreu %d vez(es) nesta Ã¡rea.\n", contagemMortes);
    if (contagemMortes >= 3) {
        printf("Suas chances acabaram... Fim de jogo.\n");
        mudarEstadoJogo(ESTADO_DERROTA);
        return;
    }
    printf("Tentando de novo...\n");
    EstadoJogo estadoParaRecarregar = estadoAtual;
    botaoAtivo = 0; 
    temChave = 0; 
    reiniciarMonstros(); 
    switch (estadoParaRecarregar) {
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
        default:
            mudarEstadoJogo(ESTADO_MENU);
            return;
    }
    inicializarJogador();
    int tamanhoMapa = 0;
    if(estadoParaRecarregar == ESTADO_VILA) tamanhoMapa = TAMANHO_VILA;
    else if(estadoParaRecarregar == ESTADO_FASE1) tamanhoMapa = TAMANHO_FASE1;
    else if(estadoParaRecarregar == ESTADO_FASE2) tamanhoMapa = TAMANHO_FASE2;
    else if(estadoParaRecarregar == ESTADO_FASE3) tamanhoMapa = TAMANHO_FASE3;
    definirTileMapa(linhaJogador, colunaJogador, JOGADOR, tamanhoMapa);
    estadoAtual = estadoParaRecarregar; 
}
void morteJogador(void) {
    printf("\n----------------------------------------\n");
    printf("    VocÃª nÃ£o sobreviveu... Que pena!    ");
    printf("\n----------------------------------------\n");
    printf("Pressione Enter para tentar novamente...");
    getchar(); 
    getchar(); 
    reiniciarFase();
}
int jogoTerminou(void) {
    return !jogoRodando;
}
void limparTela(void) {
    system("clear"); 
}
void limparJogo(void) {
    printf("\nJogo encerrado. AtÃ© a prÃ³xima aventura!\n");
}
void inicializarJogador(void) {
    linhaJogador = 1;
    colunaJogador = 1;
}
int podeMoverPara(int linha, int coluna, int tamanhoMapa) {
    char tile = obterTileMapa(linha, coluna, tamanhoMapa);
    if (tile == PAREDE || tile == PORTA_FECHADA) {
        return 0; 
    }
    return 1; 
}
void moverJogador(int novaLinha, int novaColuna, int tamanhoMapa) {
    char tileAntigo = obterTileMapaBase(linhaJogador, colunaJogador, tamanhoMapa);
    definirTileMapa(linhaJogador, colunaJogador, tileAntigo, tamanhoMapa);
    linhaJogador = novaLinha;
    colunaJogador = novaColuna;
    char tileNovo = obterTileMapa(linhaJogador, colunaJogador, tamanhoMapa);
    if (tileNovo == CHAVE) {
        temChave = 1;
        printf("\nOpa! Uma chave! Deve servir para alguma porta por aqui.\n");
        mapaBase[linhaJogador][colunaJogador] = VAZIO;
    } else if (tileNovo == BOTAO) {
        printf("\n*Click* VocÃª pisou em um botÃ£o no chÃ£o.\n");
    } else if (tileNovo == ESPINHO) {
        printf("\nCuidado onde pisa! Espinhos!\n");
    } else if (tileNovo == TELETRANSPORTE) {
        printf("\nWhoosh! VocÃª entrou num portal de teletransporte!\n");
        int destLinha, destColuna;
        encontrarDestinoTeletransporte(linhaJogador, colunaJogador, &destLinha, &destColuna, tamanhoMapa);
        linhaJogador = destLinha;
        colunaJogador = destColuna;
        printf("E saiu em (%d, %d)!\n", linhaJogador, colunaJogador);
        tileNovo = obterTileMapa(linhaJogador, colunaJogador, tamanhoMapa);
    } else if (tileNovo == MONSTRO_NIVEL1 || tileNovo == MONSTRO_NIVEL2) {
        printf("\nUm monstro! RÃ¡pido!\n");
    }
    definirTileMapa(linhaJogador, colunaJogador, JOGADOR, tamanhoMapa);
}
void interagirComTile(int linha, int coluna, int tamanhoMapa) {
    if (obterTileMapaBase(linha, coluna, tamanhoMapa) == BOTAO) {
        printf("\nVocÃª pisa com forÃ§a no botÃ£o!\n");
        ativarBotao(tamanhoMapa); 
        printf("Algo parece ter mudado no ambiente...\n");
        return;
    }
    if (temChave) {
        int adjLinha[] = {linha-1, linha+1, linha, linha};
        int adjColuna[] = {coluna, coluna, coluna-1, coluna+1};
        for (int i = 0; i < 4; i++) {
            int l = adjLinha[i];
            int c = adjColuna[i];
            if (l >= 0 && l < tamanhoMapa && c >= 0 && c < tamanhoMapa && 
                obterTileMapa(l, c, tamanhoMapa) == PORTA_FECHADA) {
                printf("\nVocÃª usa a chave... *Click!* A porta destrancou!\n");
                abrirTodasPortas(tamanhoMapa); 
                temChave = 0; 
                return; 
            }
        }
    }
    int adjLinhaNPC[] = {linha-1, linha+1, linha, linha};
    int adjColunaNPC[] = {coluna, coluna, coluna-1, coluna+1};
    for (int i = 0; i < MAX_PERSONAGENS; i++) {
        if (personagens[i].ativo) { 
            for (int j = 0; j < 4; j++) {
                int l = adjLinhaNPC[j];
                int c = adjColunaNPC[j];
                if (l >= 0 && l < tamanhoMapa && c >= 0 && c < tamanhoMapa && 
                    personagens[i].linha == l && personagens[i].coluna == c) {
                    printf("\n[%c] diz: %s\n", PERSONAGEM, personagens[i].mensagem);
                    return; 
                }
            }
        }
    }
    printf("\nVocÃª olha ao redor, mas nÃ£o encontra nada para interagir aqui.\n");
}
void inicializarMapas(void) {
    for (int i = 0; i < MAX_PERSONAGENS; i++) {
        personagens[i].ativo = 0;
    }
    numTeletransportes = 0; 
    botaoAtivo = 0; 
}
void carregarMapaVila(void) {
    int tam = TAMANHO_VILA;
    for (int i = 0; i < tam; i++) {
        for (int j = 0; j < tam; j++) {
            mapaVila[i][j] = VAZIO;
            mapaBase[i][j] = VAZIO;
        }
    }
    for (int i = 0; i < tam; i++) {
        mapaVila[0][i] = mapaBase[0][i] = PAREDE;
        mapaVila[tam-1][i] = mapaBase[tam-1][i] = PAREDE;
        mapaVila[i][0] = mapaBase[i][0] = PAREDE;
        mapaVila[i][tam-1] = mapaBase[i][tam-1] = PAREDE;
    }
    mapaVila[3][3] = mapaBase[3][3] = PAREDE;
    mapaVila[3][4] = mapaBase[3][4] = PAREDE;
    mapaVila[3][5] = mapaBase[3][5] = PAREDE;
    mapaVila[3][6] = mapaBase[3][6] = PAREDE;
    mapaVila[7][2] = mapaBase[7][2] = PAREDE;
    mapaVila[7][3] = mapaBase[7][3] = PAREDE;
    mapaVila[7][4] = mapaBase[7][4] = PAREDE;
    mapaVila[6][4] = mapaBase[6][4] = PAREDE;
    mapaVila[2][2] = mapaBase[2][2] = CHAVE;
    mapaVila[8][8] = mapaBase[8][8] = PORTA_FECHADA;
    personagens[0].linha = 2; personagens[0].coluna = 7; personagens[0].ativo = 1;
    strcpy(personagens[0].mensagem, "OlÃ¡, aventureiro! Parece perdido? Use W,A,S,D pra andar e 'I' pra interagir com coisas.");
    personagens[1].linha = 5; personagens[1].coluna = 5; personagens[1].ativo = 1;
    strcpy(personagens[1].mensagem, "Ouvi dizer que tem uma chave por aqui... talvez abra aquela porta estranha?");
    personagens[2].linha = 7; personagens[2].coluna = 7; personagens[2].ativo = 1;
    strcpy(personagens[2].mensagem, "Cuidado lÃ¡ dentro! Dizem que as masmorras estÃ£o cheias de perigos...");
    for (int i = 0; i < MAX_PERSONAGENS; i++) {
        if (personagens[i].ativo) {
            mapaVila[personagens[i].linha][personagens[i].coluna] = PERSONAGEM;
            mapaBase[personagens[i].linha][personagens[i].coluna] = PERSONAGEM; 
        }
    }
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
    botaoAtivo = 0; 
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
        if (i != 10) { 
            mapaFase2[10][i] = mapaBase[10][i] = PAREDE;
        }
    }
    mapaFase2[4][4] = mapaBase[4][4] = ESPINHO;
    mapaFase2[4][15] = mapaBase[4][15] = ESPINHO;
    mapaFase2[15][4] = mapaBase[15][4] = ESPINHO;
    mapaFase2[15][15] = mapaBase[15][15] = ESPINHO;
    mapaFase2[8][12] = mapaBase[8][12] = ESPINHO;
    mapaFase2[12][8] = mapaBase[12][8] = ESPINHO;
    mapaFase2[18][10] = mapaBase[18][10] = BOTAO;
    if (botaoAtivo) { 
        mapaBase[5][5] = VAZIO; mapaFase2[5][5] = VAZIO;
        mapaBase[5][14] = VAZIO; mapaFase2[5][14] = VAZIO;
        mapaBase[14][5] = VAZIO; mapaFase2[14][5] = VAZIO;
        mapaBase[14][14] = VAZIO; mapaFase2[14][14] = VAZIO;
    } else { 
        mapaBase[5][5] = ESPINHO; mapaFase2[5][5] = ESPINHO;
        mapaBase[5][14] = ESPINHO; mapaFase2[5][14] = ESPINHO;
        mapaBase[14][5] = ESPINHO; mapaFase2[14][5] = ESPINHO;
        mapaBase[14][14] = ESPINHO; mapaFase2[14][14] = ESPINHO;
    }
    mapaFase2[3][3] = mapaBase[3][3] = CHAVE;
    mapaFase2[18][18] = mapaBase[18][18] = PORTA_FECHADA;
    reiniciarMonstros(); 
    inicializarMonstro(0, 7, 7, 0); 
}
void carregarMapaFase3(void) {
    int tam = TAMANHO_FASE3;
    numTeletransportes = 0; 
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
    for (int i = 5; i < tam-5; i += 10) {
        for (int j = 1; j < tam-1; j++) {
            if (j != 20) { 
                mapaFase3[i][j] = mapaBase[i][j] = PAREDE;
            }
        }
    }
    for (int i = 1; i < tam-1; i++) {
        for (int j = 10; j < tam-10; j += 15) {
            if (i % 10 != 5) { 
                mapaFase3[i][j] = mapaBase[i][j] = PAREDE;
            }
        }
    }
    mapaFase3[10][15] = mapaBase[10][15] = ESPINHO;
    mapaFase3[15][30] = mapaBase[15][30] = ESPINHO;
    mapaFase3[25][10] = mapaBase[25][10] = ESPINHO;
    mapaFase3[30][25] = mapaBase[30][25] = ESPINHO;
    mapaFase3[15][15] = mapaBase[15][15] = BOTAO;
    mapaFase3[5][35] = mapaBase[5][35] = TELETRANSPORTE;
    posicoesTeletransporte[numTeletransportes][0] = 5;
    posicoesTeletransporte[numTeletransportes][1] = 35;
    numTeletransportes++;
    mapaFase3[35][5] = mapaBase[35][5] = TELETRANSPORTE;
    posicoesTeletransporte[numTeletransportes][0] = 35;
    posicoesTeletransporte[numTeletransportes][1] = 5;
    numTeletransportes++;
    if (botaoAtivo) { 
        mapaFase3[20][20] = VAZIO;
        mapaBase[20][20] = VAZIO;
    } else { 
        mapaFase3[20][20] = ESPINHO;
        mapaBase[20][20] = ESPINHO;
    }
    mapaFase3[20][35] = mapaBase[20][35] = CHAVE;
    mapaFase3[5][5] = mapaBase[5][5] = PORTA_FECHADA;
    reiniciarMonstros();
    inicializarMonstro(0, 10, 10, 0); 
    inicializarMonstro(1, 30, 30, 1); 
}
char obterTileMapa(int linha, int coluna, int tamanhoMapa) {
    if (linha < 0 || coluna < 0 || linha >= tamanhoMapa || coluna >= tamanhoMapa) {
        return PAREDE; 
    }
    switch (estadoAtual) {
        case ESTADO_VILA: return mapaVila[linha][coluna];
        case ESTADO_FASE1: return mapaFase1[linha][coluna];
        case ESTADO_FASE2: return mapaFase2[linha][coluna];
        case ESTADO_FASE3: return mapaFase3[linha][coluna];
        default: return VAZIO; 
    }
}
char obterTileMapaBase(int linha, int coluna, int tamanhoMapa) {
    if (linha < 0 || coluna < 0 || linha >= tamanhoMapa || coluna >= tamanhoMapa) {
        return PAREDE;
    }
    return mapaBase[linha][coluna];
}
void definirTileMapa(int linha, int coluna, char tile, int tamanhoMapa) {
    if (linha < 0 || coluna < 0 || linha >= tamanhoMapa || coluna >= tamanhoMapa) {
        return; 
    }
    switch (estadoAtual) {
        case ESTADO_VILA: mapaVila[linha][coluna] = tile; break;
        case ESTADO_FASE1: mapaFase1[linha][coluna] = tile; break;
        case ESTADO_FASE2: mapaFase2[linha][coluna] = tile; break;
        case ESTADO_FASE3: mapaFase3[linha][coluna] = tile; break;
        default: break; 
    }
}
void desenharMapa(int tamanhoMapa) {
    printf("\nMapa Atual:\n");
    for (int i = 0; i < tamanhoMapa; i++) {
        printf("  "); 
        for (int j = 0; j < tamanhoMapa; j++) {
            printf("%c ", obterTileMapa(i, j, tamanhoMapa)); 
        }
        printf("\n"); 
    }
    printf("PosiÃ§Ã£o: (%d, %d) | Chave: %s | Vidas: %d/3\n", 
           linhaJogador, colunaJogador, 
           temChave ? "Sim" : "NÃ£o", 
           3 - contagemMortes);
}
void abrirTodasPortas(int tamanhoMapa) {
    for (int i = 0; i < tamanhoMapa; i++) {
        for (int j = 0; j < tamanhoMapa; j++) {
            if (obterTileMapa(i, j, tamanhoMapa) == PORTA_FECHADA) {
                definirTileMapa(i, j, PORTA_ABERTA, tamanhoMapa); 
                mapaBase[i][j] = PORTA_ABERTA; 
            }
        }
    }
}
void ativarBotao(int tamanhoMapa) {
    botaoAtivo = !botaoAtivo; 
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
            return;
    }
    linhaJogador = lJog;
    colunaJogador = cJog;
    definirTileMapa(linhaJogador, colunaJogador, JOGADOR, tamanhoMapa);
}
void encontrarDestinoTeletransporte(int linhaOrigem, int colunaOrigem, int* linhaDestino, int* colunaDestino, int tamanhoMapa) {
    int indiceOrigem = -1;
    for (int i = 0; i < numTeletransportes; i++) {
        if (posicoesTeletransporte[i][0] == linhaOrigem && posicoesTeletransporte[i][1] == colunaOrigem) {
            indiceOrigem = i;
            break;
        }
    }
    if (indiceOrigem == -1) {
        *linhaDestino = linhaOrigem;
        *colunaDestino = colunaOrigem;
        return;
    }
    int indiceDestino = (indiceOrigem % 2 == 0) ? indiceOrigem + 1 : indiceOrigem - 1;
    if (indiceDestino < 0 || indiceDestino >= numTeletransportes) {
        *linhaDestino = linhaOrigem; 
        *colunaDestino = colunaOrigem;
        return;
    }
    *linhaDestino = posicoesTeletransporte[indiceDestino][0];
    *colunaDestino = posicoesTeletransporte[indiceDestino][1];
}
void inicializarMonstros(void) {
    for (int i = 0; i < MAX_MONSTROS; i++) {
        monstros[i].ativo = 0;
    }
}
void inicializarMonstro(int indice, int linha, int coluna, int tipo) {
    if (indice < 0 || indice >= MAX_MONSTROS) {
        return;
    }
    monstros[indice].linha = linha;
    monstros[indice].coluna = coluna;
    monstros[indice].ativo = 1; 
    monstros[indice].tipo = tipo; 
    int tamanhoMapa = 0;
    char simboloMonstro = (tipo == 0) ? MONSTRO_NIVEL1 : MONSTRO_NIVEL2;
    switch(estadoAtual) {
        case ESTADO_FASE2: tamanhoMapa = TAMANHO_FASE2; definirTileMapa(linha, coluna, simboloMonstro, tamanhoMapa); break;
        case ESTADO_FASE3: tamanhoMapa = TAMANHO_FASE3; definirTileMapa(linha, coluna, simboloMonstro, tamanhoMapa); break;
        default: break;
    }
}
void reiniciarMonstros(void) {
    int tamanhoMapa = 0;
    switch(estadoAtual) {
        case ESTADO_FASE2: tamanhoMapa = TAMANHO_FASE2; break;
        case ESTADO_FASE3: tamanhoMapa = TAMANHO_FASE3; break;
        default: return; 
    }
    for (int i = 0; i < MAX_MONSTROS; i++) {
        if (monstros[i].ativo) {
            if (obterTileMapa(monstros[i].linha, monstros[i].coluna, tamanhoMapa) == MONSTRO_NIVEL1 ||
                obterTileMapa(monstros[i].linha, monstros[i].coluna, tamanhoMapa) == MONSTRO_NIVEL2) {
                definirTileMapa(monstros[i].linha, monstros[i].coluna, obterTileMapaBase(monstros[i].linha, monstros[i].coluna, tamanhoMapa), tamanhoMapa);
            }
            monstros[i].ativo = 0; 
        }
    }
}
void atualizarMonstrosNivel1(void) {
    int tamanhoMapa = (estadoAtual == ESTADO_FASE2) ? TAMANHO_FASE2 : TAMANHO_FASE3;
    for (int i = 0; i < MAX_MONSTROS; i++) {
        if (monstros[i].ativo && monstros[i].tipo == 0) {
            char tileBaseAntigo = obterTileMapaBase(monstros[i].linha, monstros[i].coluna, tamanhoMapa);
            definirTileMapa(monstros[i].linha, monstros[i].coluna, tileBaseAntigo, tamanhoMapa);
            int direcao = rand() % 4; 
            int novaLinha = monstros[i].linha;
            int novaColuna = monstros[i].coluna;
            switch (direcao) {
                case 0: novaLinha--; break;
                case 1: novaLinha++; break;
                case 2: novaColuna--; break;
                case 3: novaColuna++; break;
            }
            char novoTile = obterTileMapa(novaLinha, novaColuna, tamanhoMapa);
            if (novoTile != PAREDE && novoTile != PORTA_FECHADA && novoTile != PORTA_ABERTA &&
                novoTile != CHAVE && novoTile != BOTAO && novoTile != TELETRANSPORTE &&
                novoTile != MONSTRO_NIVEL1 && novoTile != MONSTRO_NIVEL2 && novoTile != PERSONAGEM &&
                novoTile != JOGADOR) { 
                monstros[i].linha = novaLinha;
                monstros[i].coluna = novaColuna;
            }
            definirTileMapa(monstros[i].linha, monstros[i].coluna, MONSTRO_NIVEL1, tamanhoMapa);
        }
    }
}
void atualizarMonstrosNivel2(void) {
    int tamanhoMapa = TAMANHO_FASE3; 
    if (estadoAtual != ESTADO_FASE3) return;
    for (int i = 0; i < MAX_MONSTROS; i++) {
        if (monstros[i].ativo && monstros[i].tipo == 1) {
            char tileBaseAntigo = obterTileMapaBase(monstros[i].linha, monstros[i].coluna, tamanhoMapa);
            definirTileMapa(monstros[i].linha, monstros[i].coluna, tileBaseAntigo, tamanhoMapa);
            int difLinha = linhaJogador - monstros[i].linha;
            int difColuna = colunaJogador - monstros[i].coluna;
            int novaLinha = monstros[i].linha;
            int novaColuna = monstros[i].coluna;
            if (abs(difLinha) > abs(difColuna)) { 
                novaLinha += (difLinha > 0) ? 1 : -1;
            } else if (abs(difColuna) > abs(difLinha)) { 
                novaColuna += (difColuna > 0) ? 1 : -1;
            } else if (difLinha != 0) { 
                 novaLinha += (difLinha > 0) ? 1 : -1;
            } else if (difColuna != 0) { 
                 novaColuna += (difColuna > 0) ? 1 : -1;
            } 
            char novoTile = obterTileMapa(novaLinha, novaColuna, tamanhoMapa);
            int podeMoverDireto = (novoTile != PAREDE && novoTile != PORTA_FECHADA && novoTile != PORTA_ABERTA &&
                                   novoTile != CHAVE && novoTile != BOTAO && novoTile != TELETRANSPORTE &&
                                   novoTile != MONSTRO_NIVEL1 && novoTile != MONSTRO_NIVEL2 && novoTile != PERSONAGEM &&
                                   novoTile != JOGADOR);
            if (podeMoverDireto) {
                monstros[i].linha = novaLinha;
                monstros[i].coluna = novaColuna;
            } else {
                novaLinha = monstros[i].linha; 
                novaColuna = monstros[i].coluna;
                if (abs(difLinha) > abs(difColuna)) { 
                    novaColuna += (difColuna > 0) ? 1 : -1;
                } else { 
                    novaLinha += (difLinha > 0) ? 1 : -1;
                }
                novoTile = obterTileMapa(novaLinha, novaColuna, tamanhoMapa);
                int podeMoverAlternativa = (novoTile != PAREDE && novoTile != PORTA_FECHADA && novoTile != PORTA_ABERTA &&
                                           novoTile != CHAVE && novoTile != BOTAO && novoTile != TELETRANSPORTE &&
                                           novoTile != MONSTRO_NIVEL1 && novoTile != MONSTRO_NIVEL2 && novoTile != PERSONAGEM &&
                                           novoTile != JOGADOR);
                if (podeMoverAlternativa) {
                    monstros[i].linha = novaLinha;
                    monstros[i].coluna = novaColuna;
                } else {
                }
            }
            definirTileMapa(monstros[i].linha, monstros[i].coluna, MONSTRO_NIVEL2, tamanhoMapa);
        }
    }
}
void verificarColisaoJogadorMonstro(int tamanhoMapa) {
    for (int i = 0; i < MAX_MONSTROS; i++) {
        if (monstros[i].ativo && 
            monstros[i].linha == linhaJogador && 
            monstros[i].coluna == colunaJogador) {
            printf("\nArgh! VocÃª foi pego por um %s!\n", (monstros[i].tipo == 0) ? "Monstro (X)" : "Monstro (V)");
            morteJogador(); 
            return; 
        }
    }
}
void desenharJogo(void) {
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
            break;
        case ESTADO_VILA:
            printf("\n--- Explorando a Vila ---\n");
            desenharMapa(TAMANHO_VILA);
            break;
        case ESTADO_FASE1:
            printf("\n--- Masmorra - NÃ­vel 1 ---\n");
            desenharMapa(TAMANHO_FASE1);
            break;
        case ESTADO_FASE2:
            printf("\n--- Masmorra - NÃ­vel 2 ---\n");
            desenharMapa(TAMANHO_FASE2);
            break;
        case ESTADO_FASE3:
            printf("\n--- Masmorra - NÃ­vel 3 (Final) ---\n");
            desenharMapa(TAMANHO_FASE3);
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
    printf("*                                                *\n");
    printf("*           BEM-VINDO AO DUNGEON CRAWLER         *\n");
    printf("*                                                *\n");
    printf("**************************************************\n");
    printf("\n                O que vocÃª deseja fazer?\n");
    printf("\n                  [1] Iniciar Aventura
");
    printf("                  [2] Ver CrÃ©ditos
");
    printf("                  [3] Desistir (Sair)
");
    printf("\n                Digite o nÃºmero da opÃ§Ã£o: ");
}
void desenharCreditos(void) {
    printf("\n=================== CRÃ‰DITOS ===================\n");
    printf("\n    Este humilde jogo foi criado por:
");
    printf("        > Athos Ramos
");
    printf("        > Bruno Henrique
");
    printf("        > Josiel Teixeira
");
    printf("\n    Inspirado em clÃ¡ssicos jogos de masmorra.
");
    printf("\n==================================================\n");
    printf("\nPressione Enter para voltar ao Menu Principal...");
}
void desenharVitoria(void) {
    printf("\n##################################################\n");
    printf("#                                                #\n");
    printf("#                   VITÃ“RIA!!!                   #\n");
    printf("#                                                #\n");
    printf("##################################################\n");
    printf("\n    ParabÃ©ns, grande aventureiro(a)!
");
    printf("    VocÃª desbravou as profundezas da masmorra
");
    printf("    e superou todos os desafios! IncrÃ­vel!
");
    printf("\n    O tesouro (imaginÃ¡rio) Ã© todo seu!
");
    printf("\n##################################################\n");
    printf("\nPressione Enter para retornar ao Menu Principal...");
}
void desenharDerrota(void) {
    printf("\n++++++++++++++++++++++++++++++++++++++++++++++++++\n");
    printf("+                                                +
");
    printf("+                 FIM DE JOGO                  +
");
    printf("+                                                +
");
    printf("++++++++++++++++++++++++++++++++++++++++++++++++++\n");
    printf("\n    Ah, que pena... A masmorra levou a melhor.
");
    printf("    VocÃª nÃ£o conseguiu completar sua missÃ£o desta vez.
");
    printf("\n    Mas nÃ£o desanime! Quem sabe na prÃ³xima?
");
    printf("\n++++++++++++++++++++++++++++++++++++++++++++++++++\n");
    printf("\nPressione Enter para retornar ao Menu Principal...");
}
void processarEntradaMenu(char entrada) {
    switch (entrada) {
        case '1':
            printf("\nIniciando a aventura! Prepare-se...\n");
            mudarEstadoJogo(ESTADO_VILA); 
            break;
        case '2':
            mudarEstadoJogo(ESTADO_CREDITOS);
            break;
        case '3':
            mudarEstadoJogo(ESTADO_SAIR);
            jogoRodando = 0; 
            break;
        default:
            printf("\nOpÃ§Ã£o invÃ¡lida. Por favor, escolha 1, 2 ou 3.\n");
            printf("Pressione Enter para continuar...");
            getchar(); 
            getchar(); 
            break;
    }
}
void processarEntradaCreditos(char entrada) {
    mudarEstadoJogo(ESTADO_MENU);
}
