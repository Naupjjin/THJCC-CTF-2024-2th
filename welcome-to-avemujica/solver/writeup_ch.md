# 🎭🎭🎭🎭🎭Welcome to AVE Mujica🎶
> Author: 堇姬Naup

tag: `pwn`

## 分析
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>

void print_Sakiko() {
    FILE *file;
    char line[0x100];  


    file = fopen("ave_mujica.txt", "r");

    if (file == NULL) {
        printf("CRYCHIC Funeral\n");
        return 1;
    }

    while (fgets(line, sizeof(line), file)) {
        printf("%s", line);
    }
    fclose(file);
}

int main() {
    void *stage;
    int AVEmujica;
    char your_input[16];

    print_Sakiko();
    
    AVEmujica = open("flag", O_RDONLY);
    if (AVEmujica < 0) {
        printf("…ようこそ。Ave Mujica の世界へ\n");
        exit(0);
    }

    stage = malloc(0x100);
    read(AVEmujica, stage, 0x100);
    printf("Where is AVE Mujica: %p\n", stage);
    printf("呪縛なの？\n救済なの？ ");
    gets(your_input);
    printf("Ave Musica…戻れない所まで\n");

    return 0;
}
```

他malloc一塊chunk並存放slime core(flag)
並且使用者可以輸入一些東西，這裡存在無長度限制的buffer overflow					  
					 
## 攻擊					   
有buffer overflow了但是會碰到很多問題，首先是保護全開
再來是可以寫ret address但是不知道要寫甚麼，ROPgadget不夠，又沒辦法leaklibc					   
					   
我們只有flag的位置而已
					   
![stack_smashing](img/stack_smashing.png)

這邊觀察一件事，當我們在glibc 2.23觸發stack smashing detect的時候，他會噴出./demo，也就是你ELF的位置			
					   
他是去哪裡找到ELF的路徑的					   
					   
接下來來翻source code					   

通常如果有開canary會在最下方看到`__stack_chk_fail`	
追進去看會call `__fortify_fail`					   
https://elixir.bootlin.com/glibc/glibc-2.23.90/source/debug/stack_chk_fail.c
```c
void
__attribute__ ((noreturn))
__stack_chk_fail (void)
{
  __fortify_fail ("stack smashing detected");
}
```

`__fortify_fail`會call __libc_message，ELF路徑就是`__libc_argv[0]`					  
https://elixir.bootlin.com/glibc/glibc-2.23.90/source/debug/fortify_fail.c#L26
```c
void
__attribute__ ((noreturn)) internal_function
__fortify_fail (const char *msg)
{
  /* The loop is added only to keep gcc happy.  */
  while (1)
    __libc_message (2, "*** %s ***: %s terminated\n",
		    msg, __libc_argv[0] ?: "<unknown>");
}
```					   
					   
這是一個在stack上的一個pointer，這裡會印出他指向位置上的值					   
結合buffer overflow，如果我們能一直往下蓋，把該pointer蓋成flag的位置，就可以透過stack smashing去leak出flag了