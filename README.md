#include <ncurses.h>
#include <chrono>
#include <fstream>
#include <cstring>
#include <string>
#include <sstream>
#include <iostream>

using namespace std;

// Konfigurasi
// Ganti nama variabel untuk menghindari konflik dengan variabel global ncurses (COLS dan LINES)
const int MAX_ROWS = 9;
const int MAX_COLS = 20;
const int TIME_LIMIT = 60;         // detik
const int MAX_SCORES = 10;         // banyak highscore disimpan
const char HIGHSCORE_FILE[] = "highscore.txt";

// Map
// Gunakan MAX_ROWS dan MAX_COLS
char baseMaze[MAX_ROWS][MAX_COLS + 1] = {
    "####################",
    "#C.....#...........#",
    "#.####.#.####.###..#",
    "#......#......#....#",
    "####.#.######.#.####",
    "#....#....#...#....#",
    "#.########.#.####..#",
    "#..................#",
    "####################"
};

// Struktur data untuk highscore
struct ScoreEntry {
    char name[21];     // max 20 karakter + '\0'
    int  score;
};

// ---- Fungsi bantuan Highscore ----

int getBestScore(ScoreEntry scores[], int count) {
    int best = 0;
    for (int i = 0; i < count; ++i) {
        if (scores[i].score > best) best = scores[i].score;
    }
    return best;
}

void sortScores(ScoreEntry scores[], int count) {
    for (int i = 0; i < count; ++i) {
        for (int j = i + 1; j < count; ++j) {
            if (scores[j].score > scores[i].score) {
                ScoreEntry tmp = scores[i];
                scores[i] = scores[j];
                scores[j] = tmp;
            }
        }
    }
}

// Menghapus spasi di akhir (trailing spaces)
void trimTrailing(char* str) {
    int len = strlen(str);
    while (len > 0 && (str[len - 1] == ' ' || str[len - 1] == '\t')) {
        str[--len] = '\0';
    }
}

// Format file: NAMA|SKOR (nama boleh pakai spasi)
void loadHighscores(ScoreEntry scores[], int &count) {
    count = 0;
    ifstream in(HIGHSCORE_FILE);
    if (!in) return;

    string line;
    while (count < MAX_SCORES && std::getline(in, line)) {
        if (line.empty()) continue;
        size_t pos = line.find('|');
        if (pos == string::npos) continue;

        string nameStr  = line.substr(0, pos);
        string scoreStr = line.substr(pos + 1);

        // parse skor
        stringstream ss(scoreStr);
        int score = 0;
        ss >> score;
        if (ss.fail() || !ss.eof()) continue;

        // simpan ke array
        strncpy(scores[count].name, nameStr.c_str(), 20);
        scores[count].name[20] = '\0';
        trimTrailing(scores[count].name);
        if (scores[count].name[0] == '\0') continue;
        
        scores[count].score = score;
        count++;
    }
    sortScores(scores, count);
}

void saveHighscores(ScoreEntry scores[], int count) {
    ofstream out(HIGHSCORE_FILE);
    if (!out) return;

    for (int i = 0; i < count; ++i) {
        out << scores[i].name << "|" << scores[i].score << "\n";
    }
}

bool nameExists(ScoreEntry scores[], int count, const char* name) {
    for (int i = 0; i < count; ++i) {
        if (strcmp(scores[i].name, name) == 0) {
            return true;
        }
    }
    return false;
}

void addHighscore(ScoreEntry scores[], int &count, const char* name, int score) {
    if (score <= 0) return;

    ScoreEntry newEntry;
    strncpy(newEntry.name, name, 20);
    newEntry.name[20] = '\0';
    newEntry.score = score;

    if (count < MAX_SCORES) {
        scores[count] = newEntry;
        count++;
    } else {
        int minIdx = 0;
        for (int i = 1; i < count; ++i) {
            if (scores[i].score < scores[minIdx].score) {
                minIdx = i;
            }
        }
        if (score <= scores[minIdx].score) {
            return;
        }
        scores[minIdx] = newEntry;
    }

    sortScores(scores, count);
}

// ---- Fungsi tampilan maze ----

// Gunakan MAX_ROWS dan MAX_COLS
void drawMaze(char maze[MAX_ROWS][MAX_COLS + 1]) {
    for (int y = 0; y < MAX_ROWS; ++y) {
        for (int x = 0; x < MAX_COLS; ++x) {
            if (y + 1 < LINES && x + 2 < COLS) { 
                 mvaddch(y + 1, x + 2, maze[y][x]); 
            }
        }
    }
}

// Gunakan MAX_ROWS dan MAX_COLS
void resetMaze(char maze[MAX_ROWS][MAX_COLS + 1]) {
    for (int y = 0; y < MAX_ROWS; ++y) {
        strcpy(maze[y], baseMaze[y]); 
    }
}

// ---- Menu Highscore ----

void showHighscoresScreen(ScoreEntry scores[], int count) {
    clear();
    mvprintw(1, 4, "=== DAFTAR HIGHSCORE ===");

    if (count == 0) {
        mvprintw(3, 4, "Belum ada highscore tersimpan.");
    } else {
        mvprintw(3, 4, "No  Nama                       Skor");
        mvprintw(4, 4, "---------------------------------");
        for (int i = 0; i < count; ++i) {
            mvprintw(6 + i, 4, "%2d. %-20s %5d",
                     i + 1, scores[i].name, scores[i].score);
        }
    }

    mvprintw(6 + count + 2, 4, "Tekan tombol apapun untuk kembali ke menu...");
    getch();
}

// ---- Menu depan ----
// return: 0 = Mulai Game, 1 = Lihat Highscore, 2 = Keluar

int showMenu(int highScore) {
    const char* menuItems[3] = { "Mulai Game", "Lihat Highscore", "Keluar" };
    int choice = 0;
    int ch;

    nodelay(stdscr, FALSE);
    keypad(stdscr, TRUE);

    while (true) {
        clear();
        mvprintw(1, 4, "=== TERMINAL PACMAN (ncurses) ===");
        mvprintw(3, 4, "Highscore tertinggi: %d", highScore);
        mvprintw(5, 4, "Gunakan panah atas/bawah, Enter untuk pilih.");

        for (int i = 0; i < 3; ++i) {
            if (i == choice) attron(A_REVERSE);
            mvprintw(7 + i, 6, "%s", menuItems[i]);
            if (i == choice) attroff(A_REVERSE);
        }

        ch = getch();
        if (ch == KEY_UP) {
            choice = (choice - 1 + 3) % 3;
        } else if (ch == KEY_DOWN) {
            choice = (choice + 1) % 3;
        } else if (ch == '\n' || ch == '\r') {
            break;
        }
    }

    return choice;
}

// ---- Game utama ----

int playGame(int currentHighScore) {
    // Gunakan MAX_ROWS dan MAX_COLS untuk deklarasi array lokal
    char maze[MAX_ROWS][MAX_COLS + 1]; 
    resetMaze(maze);

    int playerX = 0, playerY = 0;
    int totalDots = 0;
    int eatenDots = 0;

    for (int y = 0; y < MAX_ROWS; ++y) {
        for (int x = 0; x < MAX_COLS; ++x) {
            if (maze[y][x] == 'C') {
                playerX = x;
                playerY = y;
            }
            if (maze[y][x] == '.') {
                totalDots++;
            }
        }
    }

    nodelay(stdscr, TRUE);
    keypad(stdscr, TRUE);

    auto startTime = chrono::steady_clock::now();
    bool running = true;

    while (running) {
        auto now = chrono::steady_clock::now();
        int elapsed    = (int) chrono::duration_cast<chrono::seconds>(now - startTime).count();
        int remaining = TIME_LIMIT - elapsed;

        if (remaining <= 0) {
            break;
        }

        clear();
        mvprintw(0, 4, "TERMINAL PACMAN");
        mvprintw(MAX_ROWS + 2, 4, "Score: %d / %d    Highscore: %d    Waktu: %d",
                 eatenDots, totalDots, currentHighScore, remaining);
        mvprintw(MAX_ROWS + 4, 4, "Kontrol: panah / WASD, Q untuk keluar");

        drawMaze(maze);
        refresh();

        int ch = getch();

        int newX = playerX;
        int newY = playerY;

        if (ch == 'q' || ch == 'Q') {
            running = false;
            break;
        } else if (ch == KEY_UP || ch == 'w' || ch == 'W') {
            newY--;
        } else if (ch == KEY_DOWN || ch == 's' || ch == 'S') {
            newY++;
        } else if (ch == KEY_LEFT || ch == 'a' || ch == 'A') {
            newX--;
        } else if (ch == KEY_RIGHT || ch == 'd' || ch == 'D') {
            newX++;
        } else {
            napms(50); 
            continue;
        }

        if (newX < 0 || newX >= MAX_COLS || newY < 0 || newY >= MAX_ROWS) {
            continue;
        }

        char target = maze[newY][newX];

        if (target == '#') {
            continue;
        }

        if (target == '.') {
            eatenDots++;
        }

        maze[playerY][playerX] = ' ';
        playerX = newX;
        playerY = newY;
        maze[playerY][playerX] = 'C';

        if (eatenDots == totalDots) {
            break;
        }
    }

    nodelay(stdscr, FALSE);
    return eatenDots;
}

// ---- Minta nama & simpan skor ----

void promptAndAddHighscore(ScoreEntry scores[], int &count, int score) {
    if (score <= 0) {
        return;
    }

    char name[21];

    while (true) {
        clear();
        mvprintw(2, 4, "Permainan selesai! Skor kamu: %d", score);
        mvprintw(4, 4, "Masukkan nama (boleh spasi, max 20 karakter): ");

        echo();
        curs_set(1);
        
        getnstr(name, 20); 
        
        noecho();
        curs_set(0);

        char trimmed_name[21];
        int start = 0;
        int end = strlen(name) - 1;

        while (name[start] == ' ' || name[start] == '\t') start++;
        while (end >= start && (name[end] == ' ' || name[end] == '\t')) end--;

        int len = 0;
        for (int i = start; i <= end; i++) {
            trimmed_name[len++] = name[i];
        }
        trimmed_name[len] = '\0';
        strcpy(name, trimmed_name);


        if (name[0] == '\0') {
            mvprintw(6, 4, "Nama tidak boleh kosong! Tekan tombol apapun untuk coba lagi...");
            getch();
            continue;
        }

        if (nameExists(scores, count, name)) {
            mvprintw(6, 4, "Nama '%s' sudah dipakai! Gunakan nama lain.", name);
            mvprintw(7, 4, "Tekan tombol apapun untuk coba lagi...");
            getch();
            continue;
        }

        break;
    }

    addHighscore(scores, count, name, score);
    saveHighscores(scores, count);

    mvprintw(9, 4, "Skor tersimpan dengan nama '%s'!", name);
    mvprintw(10, 4, "Tekan tombol apapun untuk kembali ke menu...");
    getch();
}

// ---- MAIN ----

int main() {
    ScoreEntry scores[MAX_SCORES];
    int scoreCount = 0;

    initscr();
    cbreak();
    noecho();
    curs_set(0);
    keypad(stdscr, TRUE);
    
    // Gunakan variabel global ncurses COLS dan LINES untuk pengecekan
    if (LINES < MAX_ROWS + 6 || COLS < MAX_COLS + 6) {
        endwin();
        std::cerr << "Ukuran terminal terlalu kecil! Minimum: " << MAX_ROWS + 6 << " baris, " << MAX_COLS + 6 << " kolom." << std::endl;
        return 1;
    }

    loadHighscores(scores, scoreCount);

    bool running = true;

    while (running) {
        int best = getBestScore(scores, scoreCount);
        int menuChoice = showMenu(best);

        if (menuChoice == 2) {
            running = false;
        } else if (menuChoice == 1) {
            showHighscoresScreen(scores, scoreCount);
        } else {
            int currentBest = getBestScore(scores, scoreCount);
            int score = playGame(currentBest);

            if (score > 0) {
                promptAndAddHighscore(scores, scoreCount, score);
            }
        }
    }

    endwin();
    return 0;
}
