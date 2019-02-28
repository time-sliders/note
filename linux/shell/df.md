-h, --human-readable  print sizes in human readable format (e.g., 1K 234M 2G)  
-l, --local        只显示本机的文件系统




    df --help
    用法：df [选项]... [文件]...
    Show information about the file system on which each FILE resides,
    or all file systems by default.
    Mandatory arguments to long options are mandatory for short options too.
      -a, --all             include dummy file systems
      -B, --block-size=SIZE  scale sizes by SIZE before printing them; e.g.,
                               '-BM' prints sizes in units of 1,048,576 bytes;
                               see SIZE format below
          --direct          show statistics for a file instead of mount point
          --total           produce a grand total
      -h, --human-readable  print sizes in human readable format (e.g., 1K 234M 2G)
      -H, --si              likewise, but use powers of 1000 not 1024
      -i, --inodes        显示inode 信息而非块使用量
      -k            即--block-size=1K
      -l, --local        只显示本机的文件系统
          --no-sync        取得使用量数据前不进行同步动作(默认)
          --output[=FIELD_LIST]  use the output format defined by FIELD_LIST,
                                   or print all fields if FIELD_LIST is omitted.
      -P, --portability     use the POSIX output format
          --sync            invoke sync before getting usage info
      -t, --type=TYPE       limit listing to file systems of type TYPE
      -T, --print-type      print file system type
      -x, --exclude-type=TYPE   limit listing to file systems not of type TYPE
      -v                    (ignored)
          --help        显示此帮助信息并退出
          --version        显示版本信息并退出
    所显示的数值是来自 --block-size、DF_BLOCK_SIZE、BLOCK_SIZE
    及 BLOCKSIZE 环境变量中第一个可用的 SIZE 单位。
    否则，默认单位是 1024 字节(或是 512，若设定 POSIXLY_CORRECT 的话)。
    SIZE is an integer and optional unit (example: 10M is 10*1024*1024).  Units
    are K, M, G, T, P, E, Z, Y (powers of 1024) or KB, MB, ... (powers of 1000).
    FIELD_LIST is a comma-separated list of columns to be included.  Valid
    field names are: 'source', 'fstype', 'itotal', 'iused', 'iavail', 'ipcent',
    'size', 'used', 'avail', 'pcent', 'file' and 'target' (see info page).
    GNU coreutils online help: <http://www.gnu.org/software/coreutils/>
    请向<http://translationproject.org/team/zh_CN.html> 报告df 的翻译错误
    要获取完整文档，请运行：info coreutils 'df invocation'
