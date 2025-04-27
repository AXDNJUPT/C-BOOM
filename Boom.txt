#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <string.h>
#include <ctype.h>
#include <wchar.h>
#include <locale.h>

#define MAX_ROWS 24
#define MAX_COLS 24
#define MAX_MINES 99
#define MAX_SCORES 5
#define MAX_NAME_LEN 50

typedef struct {
    wchar_t name[MAX_NAME_LEN];
    float time;
} HighScore;

typedef struct {
    int is_mine;
    int surrounding;
    int revealed;
    int marked;
} Cell;

// 全局游戏状态
Cell game_grid[MAX_ROWS][MAX_COLS];
wchar_t display_grid[MAX_ROWS][MAX_COLS];
int rows, cols, num_mines;
HighScore highscores[MAX_SCORES];
int score_count = 0;

// 函数声明
void init_game(int level);
void generate_mines();
void calculate_surroundings();
void print_display();
int process_input(int mark, int x, int y);
int check_win();
void save_highscore(float time, const wchar_t* name);
void load_highscores();
void display_highscores();
void reveal(int x, int y);
void print_rules();

// 输入处理宏（显眼注释）
/* 输入处理部分开始 */
#define INPUT_BUFFER_SIZE 50
int get_valid_input(int *mark, int *x, int *y) {
    wchar_t input[INPUT_BUFFER_SIZE];
    wprintf(L"输入操作（格式：标记(行,列)，例：1(2,3）标记雷，0(2,3）翻开）：");
    fgetws(input, INPUT_BUFFER_SIZE, stdin);

    // 清理输入中的空白字符
    wchar_t *clean_input = input;
    while (*clean_input && iswspace(*clean_input)) clean_input++;
    
    int args = swscanf(clean_input, L"%d(%d,%d)", mark, x, y);
    if (args != 3) return 0;
    
    // 坐标有效性检查
    if (*x < 1 || *x > rows || *y < 1 || *y > cols) return 0;
    
    return 1;
}
/* 输入处理部分结束 */

int main() {
    setlocale(LC_ALL, "");
    srand(time(NULL));
    load_highscores();

    int choice;
    do {
        wprintf(L"\n=== 扫雷游戏 ===\n");
        wprintf(L"1. 开始游戏\n2. 查看排行榜\n3. 游戏说明\n4. 退出\n请选择：");
        wscanf(L"%d", &choice);
        while (getwchar() != L'\n');

        switch(choice) {
            case 1: {
                int level;
                wprintf(L"\n选择难度：\n1. 初级(9x9, 5雷)\n2. 中级(16x16, 40雷)\n3. 高级(24x24, 99雷)\n选择：");
                wscanf(L"%d", &level);
                while (getwchar() != L'\n');

                if (level == 1) {
                    rows = cols = 9;
                    num_mines = 3;
                } else if (level == 2) {
                    rows = cols = 16;
                    num_mines = 40;
                } else if (level == 3) {
                    rows = cols = 24;
                    num_mines = 99;
                } else {
                    wprintf(L"无效选择！\n");
                    continue;
                }

                init_game(level);
                clock_t start = clock();
                int game_over = 0, win = 0;

                while (!game_over) {
                    print_display();
                    
                    int mark, x, y;
                    if (!get_valid_input(&mark, &x, &y)) {
                        wprintf(L"无效输入！请按格式输入\n");
                        continue;
                    }

                    x--; y--; // 转换为0-based索引
                    int result = process_input(mark, x, y);

                    if (result == -1) { // 触雷
                        game_over = 1;
                        // 显示所有地雷
                        for (int i = 0; i < rows; i++)
                            for (int j = 0; j < cols; j++)
                                if (game_grid[i][j].is_mine)
                                    display_grid[i][j] = L'*';
                        print_display();
                        wprintf(L"踩到地雷！游戏结束！\n");
                    } else if (check_win()) {
                        game_over = win = 1;
                    }
                }

                if (win) {
                    float duration = (float)(clock() - start) / CLOCKS_PER_SEC;
                    wprintf(L"\n用时：%.2f秒\n", duration);

                    if (score_count < MAX_SCORES || duration < highscores[MAX_SCORES-1].time) {
                        wprintf(L"进入排行榜！输入姓名：");
                        wchar_t name[MAX_NAME_LEN];
                        fgetws(name, MAX_NAME_LEN, stdin);
                        name[wcscspn(name, L"\n")] = L'\0';
                        save_highscore(duration, name);
                    }
                }
                break;
            }
            case 2:
                display_highscores();
                break;
            case 3:
                print_rules();
                break;
            case 4:
                wprintf(L"退出游戏！\n");
                break;
            default:
                wprintf(L"无效选择！\n");
        }
    } while (choice != 4);

    return 0;
}

void init_game(int level) {
    // 初始化网格
    memset(game_grid, 0, sizeof(game_grid));
    for (int i = 0; i < MAX_ROWS; i++)
        for (int j = 0; j < MAX_COLS; j++)
            display_grid[i][j] = L'.';
    
    generate_mines();
    calculate_surroundings();
}

void generate_mines() {
    int placed = 0;
    while (placed < num_mines) {
        int x = rand() % rows;
        int y = rand() % cols;
        if (!game_grid[x][y].is_mine) {
            game_grid[x][y].is_mine = 1;
            placed++;
        }
    }
}

void calculate_surroundings() {
    const int dx[] = {-1,-1,-1,0,0,1,1,1};
    const int dy[] = {-1,0,1,-1,1,-1,0,1};

    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            if (game_grid[i][j].is_mine) continue;
            
            int cnt = 0;
            for (int k = 0; k < 8; k++) {
                int nx = i + dx[k], ny = j + dy[k];
                if (nx >= 0 && nx < rows && ny >= 0 && ny < cols)
                    cnt += game_grid[nx][ny].is_mine;
            }
            game_grid[i][j].surrounding = cnt;
        }
    }
}

void print_display() {
    wprintf(L"\n  ");
    for (int j = 0; j < cols; j++)
        wprintf(L"%2d", j+1);
    wprintf(L"\n");

    for (int i = 0; i < rows; i++) {
        wprintf(L"%2d ", i+1);
        for (int j = 0; j < cols; j++) {
            if (display_grid[i][j] == L'@') wprintf(L" @");
            else if (display_grid[i][j] == L'.') wprintf(L" .");
            else wprintf(L" %lc", display_grid[i][j]);
        }
        wprintf(L"\n");
    }
}

int process_input(int mark, int x, int y) {
    if (mark == 1) { // 标记/取消标记
        if (display_grid[x][y] == L'@') {
            display_grid[x][y] = L'.';
            game_grid[x][y].marked = 0;
        } else if (display_grid[x][y] == L'.') {
            display_grid[x][y] = L'@';
            game_grid[x][y].marked = 1;
        }
        return 0;
    } else if (mark == 0) { // 翻开
        if (game_grid[x][y].is_mine) return -1;
        reveal(x, y);
        return 0;
    }
    return 0;
}

void reveal(int x, int y) {
    if (x < 0 || x >= rows || y < 0 || y >= cols) return;
    if (game_grid[x][y].revealed) return;
    if (game_grid[x][y].marked) return;

    game_grid[x][y].revealed = 1;
    if (game_grid[x][y].surrounding > 0) {
        display_grid[x][y] = L'0' + game_grid[x][y].surrounding;
    } else {
        display_grid[x][y] = L' ';
        // 递归展开相邻空白区域
        const int dx[] = {-1,-1,-1,0,0,1,1,1};
        const int dy[] = {-1,0,1,-1,1,-1,0,1};
        for (int i = 0; i < 8; i++)
            reveal(x + dx[i], y + dy[i]);
    }
}

int check_win() {
    int correct = 0;
    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            if (game_grid[i][j].is_mine) {
                if (game_grid[i][j].marked) correct++;
            } else {
                if (!game_grid[i][j].revealed) return 0;
            }
        }
    }
    return correct == num_mines;
}

void load_highscores() {
    FILE *fp = fopen("scores.txt", "r");
    if (!fp) return;

    score_count = 0;
    while (score_count < MAX_SCORES && 
          fwscanf(fp, L"%49ls %f", highscores[score_count].name, 
                  &highscores[score_count].time) == 2) {
        score_count++;
    }
    fclose(fp);
}

void save_highscore(float time, const wchar_t* name) {
    // 添加新记录
    if (score_count < MAX_SCORES) {
        wcscpy(highscores[score_count].name, name);
        highscores[score_count].time = time;
        score_count++;
    } else {
        highscores[MAX_SCORES-1] = (HighScore){ .time = time };
        wcscpy(highscores[MAX_SCORES-1].name, name);
    }

    // 冒泡排序
    for (int i = 0; i < score_count-1; i++) {
        for (int j = 0; j < score_count-1-i; j++) {
            if (highscores[j].time > highscores[j+1].time) {
                HighScore temp = highscores[j];
                highscores[j] = highscores[j+1];
                highscores[j+1] = temp;
            }
        }
    }

    // 保存到文件
    FILE *fp = fopen("scores.txt", "w");
    for (int i = 0; i < score_count && i < MAX_SCORES; i++)
        fwprintf(fp, L"%ls %.2f\n", highscores[i].name, highscores[i].time);
    fclose(fp);
}

void display_highscores() {
    wprintf(L"\n=== 排行榜 ===\n");
    for (int i = 0; i < score_count; i++)
        wprintf(L"%d. %-20ls %.2f秒\n", i+1, highscores[i].name, highscores[i].time);
}

void print_rules() {
    wprintf(L"\n=== 游戏规则 ===\n");
    wprintf(L"1. 输入格式：标记(行,列)，例如：\n");
    wprintf(L"   1(2,3) 表示标记第2行第3列为地雷\n");
    wprintf(L"   0(4,5) 表示翻开第4行第5列\n");
    wprintf(L"2. 成功标记所有地雷并翻开所有安全区域即胜利\n");
    wprintf(L"3. 错误标记或触发地雷会导致游戏失败\n");
}