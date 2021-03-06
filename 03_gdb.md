

# 03. Application Debugger  #

### 시작 전, 시간이 소요 될 수 있는 다음을 실행 해 둔다 ###
~~~
yum install debuginfo-install 
debuginfo-install procps-ng
~~~
- kdump 용 설치를 미리 해 둔다.
`yum --enablerepo=base-debuginfo install –y kernel-debuginfo-$(uname -r)`


## GDB (GNU Debugger) 개요 ##
- Strace 는 모니터링 대상의 System call 을 추적한다.
- ltrace는 모니터링 대상이 사용하는 library를 추적한다. 
- GDB는 Debugger이다. Windows의 Visual Studio 및 Eclipse에서의 Debugger mode를 생각하면 된다. 

## Application 구동 원리 ##
- Application 과 연관된 Troubleshooting을 위해서는 Application 구동 원리를 이해 해야 한다.
 - 1) application Process 구동
 - 2) Application Memory에 적재
 - 3) Application 실행 명령어 
 
- process (kernel fork)
- file system (==> ELF, 라이브러리 적재,  code/text section)
- 언제나 유사한 형태의 메모리 유지 (관리가 쉬움 ) ==> 가상메모리 사용 (물리메모리 적재는 산발/복잡)
- micro-processor 명령어 셋 실행 => RIP => code section
- RIP 실행 중에, 점프 ==> 현재상태 저장하고 다른곳으로 ==> Stack
- RIP 실행 중에, memory 생성 ==> heap 생성/적재


## core dump ##
- core file을 남기도록 설정
`ulimit -a`
~~~
[root@clu_1 ~]# ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 7235
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 7235
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
~~~

`ulimit -c unlimited`
~~~
core file size          (blocks, -c) unlimited
~~~

## normal file( not seg fault )##
wget https://raw.githubusercontent.com/windflex-sjlee/linux_troubleshooting/master/src/test_011.c

<pre>
gcc -g -o test ./test_01.c
</pre>

## debugging을 둘러보자... ##

`gdb ./test`


~~~
(gdb) list main
(gdb) bt
(gdb) print a
(gdb) print b
(gdb) disas *main
(gdb) b *main
~~~

## seg fault ##
wget https://raw.githubusercontent.com/windflex-sjlee/linux_troubleshooting/master/src/test_01.c

- `gdb ./test core xxx`

~~~
(gdb) list main
(gdb) set listsize 50
(gdb) bt
(gdb) disp p
(gdb) disas *main
(gdb) b *main
(gdb) info register   
~~~

## 좀 더 복잡한 예를 들어보자 backtrace example ##
wget https://raw.githubusercontent.com/windflex-sjlee/linux_troubleshooting/master/src/test_03.c


`gcc -g -o test_03 ./test_03.c`
`./test_03`
`Aborted (core dumped)`

## strace/ltrace 결과 비교 ##
- `starce ./test_03`
~~~
writev(3, [{"*** Error in `", 14}, {"./test_03", 9}, {"': ", 3}, {"double free or corruption (fastt"..., 35}, {": 0x", 4}, {"00000000011a4010", 16}, {" ***\n", 5}], 7*** Error in `./test_03': double free or corruption (fasttop): 0x00000000011a4010 ***
~~~
- 실행 결과로 출력된 에러메세지 정도만 알 수 있다. 
- 뭐 이건, 굳이 strace를 하지 않아도...

- `ltrace ./test_03`
~~~
[root@clu_1 ~]# ltrace ./test_03 
__libc_start_main(0x400688, 1, 0x7ffded483d68, 0x4006c0 <unfinished ...>
malloc(1)                                            = 0x611010
free(0x611010)                                       = <void>
free(0x611010*** Error in `./test_03': double free or corruption (fasttop): 0x0000000000611010 ***
~~~


## GDB 실행 with core dump ##

- `gdb ./test_03 core.xxx`

- 실행하면 아래와 같은 header 메세지를 볼 수 있다. 
~~~
Core was generated by `./test_03'.
Program terminated with signal 6, Aborted.
#0  0x00007fcf9ffa7277 in __GI_raise (sig=sig@entry=6)
    at ../nptl/sysdeps/unix/sysv/linux/raise.c:56
56	  return INLINE_SYSCALL (tgkill, 3, pid, selftid, sig);
Missing separate debuginfos, use: debuginfo-install libgcc-4.8.5-28.el7_5.1.x86_64
~~~

- `(gdb) bt`
~~~
(gdb) bt
#0  0x00007fcf9ffa7277 in __GI_raise (sig=sig@entry=6)
    at ../nptl/sysdeps/unix/sysv/linux/raise.c:56
#1  0x00007fcf9ffa8968 in __GI_abort () at abort.c:90
#2  0x00007fcf9ffe9d37 in __libc_message (do_abort=do_abort@entry=2, 
    fmt=fmt@entry=0x7fcfa00fbd58 "*** Error in `%s': %s: 0x%s ***\n")
    at ../sysdeps/unix/sysv/linux/libc_fatal.c:196
#3  0x00007fcf9fff2499 in malloc_printerr (ar_ptr=0x7fcfa0337760 <main_arena>, 
    ptr=<optimized out>, str=0x7fcfa00fbe18 "double free or corruption (fasttop)", 
    action=3) at malloc.c:5025
#4  _int_free (av=0x7fcfa0337760 <main_arena>, p=<optimized out>, have_lock=0)
    at malloc.c:3847
#5  0x0000000000400642 in Abnormal () at ./test_03.c:10
#6  0x0000000000400676 in AbnormalContainer () at ./test_03.c:19
#7  0x00000000004006a1 in main (argc=1, argv=0x7ffe48376b18) at ./test_03.c:29
~~~
- 상기 Call Stack trace는,
- main ->AbnormalContainer -> Abnormal -> init_free -> malloc_printerr ->__libc_message ->GI_abort ->GI_raise
- 순서로 함수를 실행하다가, GI_raise ==>  0x00007fcf9ffa7277 위치에서 실행 종료 되었다.
- GI_raise는 메세지를 발생하는 가젯이므로, 직접적인 원인은 malloc_printerr 이며, 원인은 double free / corruption 이다.
- 또한, 이것을 호출한 함수는 Abnormal() 이다. 

- list Abnormal 을 살펴 보자...!

~~~
(gdb) list Abnormal
1	#include <stdio.h> 
2	#include <memory.h>
3	
4	void Abnormal()
5	{
6	    int n = 1024;
7	    char *p = (char *)malloc(sizeof(char) * 1);
8	
9	    free(p);
10	    free(p);                /* double free */
~~~

- `disas /m Abnormal`
~~~
Dump of assembler code for function Abnormal:
5	{
   0x000000000040060d <+0>:	push   %rbp
   0x000000000040060e <+1>:	mov    %rsp,%rbp
   0x0000000000400611 <+4>:	sub    $0x20,%rsp

6	    int n = 1024;
   0x0000000000400615 <+8>:	movl   $0x400,-0x4(%rbp)

7	    char *p = (char *)malloc(sizeof(char) * 1);
   0x000000000040061c <+15>:	mov    $0x1,%edi
   0x0000000000400621 <+20>:	callq  0x400510 <malloc@plt>
   0x0000000000400626 <+25>:	mov    %rax,-0x10(%rbp)

8	
9	    free(p);
   0x000000000040062a <+29>:	mov    -0x10(%rbp),%rax
   0x000000000040062e <+33>:	mov    %rax,%rdi
   0x0000000000400631 <+36>:	callq  0x4004c0 <free@plt>

10	    free(p);                /* double free */
   0x0000000000400636 <+41>:	mov    -0x10(%rbp),%rax
   0x000000000040063a <+45>:	mov    %rax,%rdi
   0x000000000040063d <+48>:	callq  0x4004c0 <free@plt>

~~~

- `(gdb)info functions`
~~~
File ./test_03.c:
void Abnormal();
void AbnormalContainer();
void Normal();
int main(int, char **);
~~~

### 기타 지역변수 확인 ###
- `(gdb) info local`



## 내부 명령어 디버깅 - 내부 debug용 소스파일 다운로드 ##

~~~
yum install debuginfo-install 
debuginfo-install procps-ng
~~~

`gdb /bin/free`
`(gdb) list `

## 기존 프로세스에 gdb attach ##

~~~
ps -ef | grep http

su -
sudo yum install -y httpd
rpm -qa httpd
systemctl start httpd 
systemctl status httpd
~~~
### [참조] RPM 명렁어 요약 ###
- https://github.com/windflex-sjlee/linux_troubleshooting/blob/master/rpm_cmd_info.md

- `ps -ef | grep http`

~~~
[root@clu_1 ~]# ps -ef | grep http
root      4125     1  0 19:44 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache    4129  4125  0 19:44 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache    4130  4125  0 19:44 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache    4131  4125  0 19:44 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache    4132  4125  0 19:44 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache    4133  4125  0 19:44 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
root      4158  3749  0 19:46 pts/0    00:00:00 grep --color=auto http
~~~

- root 가 실행해준 Apache Daemon process를 잡는다. 

- `gdb /usr/sbin/httpd 4125`  or   `gdb -q -p 4125`
  - q 옵션 : quite 

~~~
(gdb) where
#0  0x00007f34334a4c53 in __select_nocancel ()
    at ../sysdeps/unix/syscall-template.S:81
#1  0x00007f3433bc05a5 in apr_sleep () from /lib64/libapr-1.so.0
#2  0x00007f3434ee5811 in ap_wait_or_timeout ()
#3  0x00007f342aaf50de in prefork_run () from /etc/httpd/modules/mod_mpm_prefork.so
#4  0x00007f3434ee4ffe in ap_run_mpm ()
#5  0x00007f3434eddd76 in main ()
~~~

### 다음과 같은 순서로 실행 중에 있다 ###
- main() -> ap_run_mpm() -> prefork_run()  -> ap_wait_or_timeout()  -> apr_sleep()  -> __select_nocancel()

~~~
(gdb) bt
(gdb) f 5
(gdb) list
~~~

- 실행중인 프로세스에서 core 파일을 생성해 준다. (이것은 Process의 스냅샷이라고 보면 된다. )
- `(gdb) generate-core-file `

### 종료시에는 기존 프로세스와 분리 시켜주자 ###
`(gdb) detach`

~~~
[root@clu_1 ~]# ls -al core*
-rw-r--r-- 1 root root 2443552 Dec  6 20:32 core.4125
-rw------- 1 root root  536576 Dec  6 20:04 core.4428
-rw------- 1 root root  536576 Dec  6 20:23 core.4983
[root@clu_1 ~]# file core.4125 
core.4125: ELF 64-bit LSB core file x86-64, version 1 (SYSV), SVR4-style, from '/usr/sbin/httpd', real uid: 0, effective uid: 0, real gid: 0, effective gid: 0, execfn: '/usr/sbin/httpd', platform: 'x86_64'
~~~
- 이제, 실제 httpd 서비스가 없어도  core파일 만으로 상태를 분석할 수 있다. 

~~~
[root@clu_1 ~]# systemctl stop httpd
[root@clu_1 ~]# gdb -q core.4125 
[New LWP 4125]
Missing separate debuginfo for the main executable file
Try: yum --enablerepo='*debug*' install /usr/lib/debug/.build-id/17/ef80493e25188af719aaf17734957e3754fc07
Core was generated by `/usr/sbin/httpd'.
#0  0x00007f34334a4c53 in ?? ()
"/root/core.4125" is a core file.
Please specify an executable to debug.
~~~





