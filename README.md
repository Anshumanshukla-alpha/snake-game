// two_snake_game_modes.c
// Includes game modes, bonus food, countdown, high score save
// Snake-Snake collision removed

#define _CRT_SECURE_NO_WARNINGS
#include <windows.h>
#include <conio.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define WIDTH  50
#define HEIGHT 22
#define MAX_LEN 512

// ---- mode configs ----
typedef struct {
    int tickSpeed;
    int bonusChance;
    int obstacleChance;
    int startingLength;
} ModeSettings;

ModeSettings MODE_CLASSIC  = {110, 150,   0, 4};
ModeSettings MODE_BATTLE   = { 75,  75,   0, 3};
ModeSettings MODE_CHAOS    = { 90, 120, 300, 4};

typedef struct { int x, y; } Point;
typedef enum { UP=0, DOWN=1, LEFT=2, RIGHT=3 } Dir;

typedef struct {
    Point body[MAX_LEN];
    int length;
    Dir dir;
    Dir nextDir;
    int alive;
    int score;
    char headChar;
    char bodyChar;
    WORD color;
} Snake;

static HANDLE hOut;

int highP1 = 0;
int highP2 = 0;


// -------------------- BASIC CONSOLE UTILS --------------------
void setCursorVisibility(int show){
    CONSOLE_CURSOR_INFO info = {20, show};
    SetConsoleCursorInfo(hOut,&info);
}

void gotoxy(int x,int y){ COORD c={x,y}; SetConsoleCursorPosition(hOut,c); }
void setColor(WORD c){ SetConsoleTextAttribute(hOut,c); }

void clearScreen(){
    CONSOLE_SCREEN_BUFFER_INFO csbi;
    DWORD count, cellCount;
    COORD home={0,0};

    GetConsoleScreenBufferInfo(hOut,&csbi);
    cellCount = csbi.dwSize.X * csbi.dwSize.Y;
    FillConsoleOutputCharacter(hOut,' ',cellCount,home,&count);
    FillConsoleOutputAttribute(hOut,csbi.wAttributes,cellCount,home,&count);
    SetConsoleCursorPosition(hOut, home);
}

void drawBorder(){
    setColor(7|8);
    for(int x=0;x<=WIDTH;x++){ gotoxy(x,0); putchar('#'); gotoxy(x,HEIGHT); putchar('#'); }
    for(int y=0;y<=HEIGHT;y++){ gotoxy(0,y); putchar('#'); gotoxy(WIDTH,y); putchar('#'); }
}

void drawText(int x,int y,WORD col,const char*s){
    setColor(col);
    gotoxy(x,y);
    fputs(s,stdout);
    setColor(7);
}

int inside(int x,int y){ return x>0 && x<WIDTH && y>0 && y<HEIGHT; }
int eq(Point a,Point b){ return (a.x==b.x && a.y==b.y); }


// -------------------- HIGH SCORE --------------------
void loadHigh(){
    FILE*f=fopen("highscore.txt","r");
    if(!f)return;
    fscanf(f,"%d%d",&highP1,&highP2);
    fclose(f);
}
void saveHigh(){
    FILE*f=fopen("highscore.txt","w");
    if(!f)return;
    fprintf(f,"%d %d", highP1, highP2);
    fclose(f);
}


// -------------------- SNAKE SYSTEM --------------------
void initSnake(Snake*s,int x,int y,Dir d,int len,char head,char body,WORD color){
    s->length=len;
    s->dir=s->nextDir=d;
    s->alive=1;
    s->score=0;
    s->headChar=head;
    s->bodyChar=body;
    s->color=color;

    for(int i=0;i<len;i++){
        s->body[i].x = x - (d==RIGHT?i:(d==LEFT?-i:0));
        s->body[i].y = y - (d==DOWN?i:(d==UP?-i:0));
    }
}

int selfHit(const Snake*s,Point h){
    for(int i=1;i<s->length;i++)
        if(eq(s->body[i],h)) return 1;
    return 0;
}

int snakeOccupies(Point p,const Snake*s){
    for(int i=0;i<s->length;i++)
        if(eq(s->body[i],p)) return 1;
    return 0;
}

int occupied(Point p,const Snake*a,const Snake*b,int obstacleCount,Point obstacles[]){
    for(int i=0;i<a->length;i++) if(eq(a->body[i],p)) return 1;
    for(int i=0;i<b->length;i++) if(eq(b->body[i],p)) return 1;
    for(int i=0;i<obstacleCount;i++) if(eq(obstacles[i],p)) return 1;
    return 0;
}

Point spawnFree(const Snake*a,const Snake*b,int obstacleCount,Point obs[]){
    Point p;
    do{
        p.x=1+rand()%(WIDTH-1);
        p.y=1+rand()%(HEIGHT-1);
    }while(occupied(p,a,b,obstacleCount,obs));
    return p;
}

Point nextPoint(Point p,Dir d){
    if(d==UP)p.y--;
    else if(d==DOWN)p.y++;
    else if(d==LEFT)p.x--;
    else p.x++;
    return p;
}

int opposite(Dir a,Dir b){
    return (a==UP&&b==DOWN)||(a==DOWN&&b==UP)||(a==LEFT&&b==RIGHT)||(a==RIGHT&&b==LEFT);
}

void erasePoint(Point p){
    gotoxy(p.x,p.y);
    putchar(' ');
}

void drawPoint(Point p,WORD col,char ch){
    setColor(col);
    gotoxy(p.x,p.y);
    putchar(ch);
    setColor(7);
}

void renderSnake(const Snake*s){
    if(!s->alive) return;
    setColor(s->color);
    for(int i=1;i<s->length;i++){
        gotoxy(s->body[i].x,s->body[i].y);
        putchar(s->bodyChar);
    }
    gotoxy(s->body[0].x,s->body[0].y);
    putchar(s->headChar);
    setColor(7);
}


// -------------------- HUD --------------------
void hud(const Snake*p1,const Snake*p2){
    char t[200];
    sprintf(t,"P1:%3d (%s) | P2:%3d (%s)   High[%d/%d]   (P=pause, Q=quit)",
            p1->score,p1->alive?"ALV":"DED",
            p2->score,p2->alive?"ALV":"DED",
            highP1, highP2);
    drawText(0,HEIGHT+1,14,t);
}


// -------------------- INPUT --------------------
void readInput(Snake*p1,Snake*p2,int*run,int*pause){
    while(_kbhit()){
        int c=_getch();
        if(c==224){
            c=_getch();
            if(p2->alive){
                if(c==72 && p2->dir!=DOWN)p2->nextDir=UP;
                else if(c==80 && p2->dir!=UP)p2->nextDir=DOWN;
                else if(c==75 && p2->dir!=RIGHT)p2->nextDir=LEFT;
                else if(c==77 && p2->dir!=LEFT)p2->nextDir=RIGHT;
            }
        }else{
            if(c>='A'&&c<='Z')c+=32;
            if(c=='q')*run=0;
            else if(c=='p')*pause=!(*pause);
            else if(p1->alive){
                if(c=='w'&&p1->dir!=DOWN)p1->nextDir=UP;
                else if(c=='s'&&p1->dir!=UP)p1->nextDir=DOWN;
                else if(c=='a'&&p1->dir!=RIGHT)p1->nextDir=LEFT;
                else if(c=='d'&&p1->dir!=LEFT)p1->nextDir=RIGHT;
            }
        }
    }
}


// -------------------- COUNTDOWN --------------------
void countdown(){
    const char*msg[]={"READY...","3","2","1","GO!"};
    for(int i=0;i<5;i++){
        drawText(WIDTH/2-4, HEIGHT/2, 14, msg[i]);
        Sleep(700);
        drawText(WIDTH/2-4, HEIGHT/2, 7, "        ");
    }
}


// -------------------- GAME OVER --------------------
int showGameOver(Snake*p1,Snake*p2){
    gotoxy(2,HEIGHT/2);
    setColor(12|8);

    if(p1->alive==0 && p2->alive==0)
        printf("DRAW!  P1:%d  P2:%d  (R=Restart)",p1->score,p2->score);
    else if(!p1->alive)
        printf("PLAYER 2 WINS!  P1:%d  P2:%d  (R=Restart)",p1->score,p2->score);
    else
        printf("PLAYER 1 WINS!  P1:%d  P2:%d  (R=Restart)",p1->score,p2->score);

    setColor(7);

    if(p1->score > highP1) highP1 = p1->score;
    if(p2->score > highP2) highP2 = p2->score;
    saveHigh();

    int k=_getch();
    return (k=='r'||k=='R');
}


// -------------------- MODE SELECTION --------------------
ModeSettings chooseMode(){
    clearScreen();
    drawText(10,5,14,"SELECT GAME MODE:");
    drawText(10,7,11,"1) CLASSIC  - normal gameplay");
    drawText(10,8,11,"2) BATTLE   - faster, more food");
    drawText(10,9,11,"3) CHAOS    - obstacles spawn randomly");
    drawText(10,11,10,"Press 1,2,3:");

    int mode=0;
    while(!mode){
        int k=_getch();
        if(k=='1') return MODE_CLASSIC;
        if(k=='2') return MODE_BATTLE;
        if(k=='3') return MODE_CHAOS;
    }
    return MODE_CLASSIC;
}


// -------------------- MAIN --------------------
int main(){
    hOut=GetStdHandle(STD_OUTPUT_HANDLE);
    srand(time(NULL));
    loadHigh();

    setCursorVisibility(0);

    while(1){
        ModeSettings mode = chooseMode();

        clearScreen();
        drawBorder();

        Snake p1,p2;
        initSnake(&p1,WIDTH/4,HEIGHT/2,RIGHT,mode.startingLength,'O','o',10);
        initSnake(&p2,3*WIDTH/4,HEIGHT/2,LEFT, mode.startingLength,'@','+',9);

        Point obstacles[200];
        int obstacleCount = 0;

        Point food = spawnFree(&p1,&p2, obstacleCount, obstacles);
        drawPoint(food,12,'*');

        int bonusActive=0;
        Point bonusFood;

        countdown();

        int run=1,pause=0;

        while(run){
            DWORD start = GetTickCount();

            readInput(&p1,&p2,&run,&pause);
            if(!run) break;
            if(pause){ Sleep(80); continue; }

            if(p1.alive && !opposite(p1.dir,p1.nextDir)) p1.dir=p1.nextDir;
            if(p2.alive && !opposite(p2.dir,p2.nextDir)) p2.dir=p2.nextDir;

            Point n1 = nextPoint(p1.body[0],p1.dir);
            Point n2 = nextPoint(p2.body[0],p2.dir);

            // No snake-snake collision: they can pass through each other

            // ----- P1 -----
            if(p1.alive){
                if(!inside(n1.x,n1.y) || selfHit(&p1,n1)){
                    p1.alive=0;
                }else{
                    int grow = eq(n1,food) || (bonusActive && eq(n1,bonusFood));

                    if(grow){
                        if(bonusActive && eq(n1,bonusFood)){ p1.score+=25; bonusActive=0; }
                        else p1.score+=10;
                        p1.length++;
                    }else erasePoint(p1.body[p1.length-1]);

                    for(int i=p1.length-1;i>0;i--) p1.body[i]=p1.body[i-1];
                    p1.body[0]=n1;
                }
            }

            // ----- P2 -----
            if(p2.alive){
                if(!inside(n2.x,n2.y) || selfHit(&p2,n2)){
                    p2.alive=0;
                }else{
                    int grow = eq(n2,food) || (bonusActive && eq(n2,bonusFood));

                    if(grow){
                        if(bonusActive && eq(n2,bonusFood)){ p2.score+=25; bonusActive=0; }
                        else p2.score+=10;
                        p2.length++;
                    }else erasePoint(p2.body[p2.length-1]);

                    for(int i=p2.length-1;i>0;i--) p2.body[i]=p2.body[i-1];
                    p2.body[0]=n2;
                }
            }

            // New food spawn
            if((p1.alive && eq(p1.body[0],food)) ||
               (p2.alive && eq(p2.body[0],food))){
                food = spawnFree(&p1,&p2, obstacleCount, obstacles);
            }

            drawPoint(food,12,'*');

            // Bonus food spawn
            if(!bonusActive && rand()%mode.bonusChance==0){
                bonusFood = spawnFree(&p1,&p2, obstacleCount, obstacles);
                bonusActive = 1;
            }
            if(bonusActive) drawPoint(bonusFood,11,'$');

            // Obstacles (CHAOS mode)
            if(mode.obstacleChance > 0 && rand()%mode.obstacleChance==0){
                obstacles[obstacleCount] = spawnFree(&p1,&p2, obstacleCount, obstacles);
                drawPoint(obstacles[obstacleCount],8,'#');
                obstacleCount++;
            }

            // Draw snakes
            renderSnake(&p1);
            renderSnake(&p2);
            hud(&p1,&p2);

            if(!p1.alive || !p2.alive){
                if(showGameOver(&p1,&p2)) goto restart_loop;
                else goto exit_loop;
            }

            DWORD elapsed = GetTickCount()-start;
            if(elapsed < mode.tickSpeed)
                Sleep(mode.tickSpeed - elapsed);
        }

        restart_loop:
            continue;

        exit_loop:
            break;
    }

    setCursorVisibility(1);
    return 0;
}
