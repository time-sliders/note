**超链接，为 source_file 创建一个名字叫 target_file 的 link 文件**

    usage: ln [-Ffhinsv] source_file [target_file]
           ln [-Ffhinsv] source_file ... target_dir
           link source_file target_file
    source_file 指的是真实的文件
    target_file 指的是要创建 link 文件
    
链接有两种，一种被称为硬链接（Hard Link），另一种被称为符号链接（Symbolic Link）。

建立硬链接时，链接文件和被链接文件必须位于同一个文件系统中，并且不能建立指向 source_file 为目录的硬链接。

而对符号链接，则不存在这个问题。  
默认情况下，ln产生硬链接。

在硬链接的情况下，参数中的 source_file 被链接至 target_file。
如果 target_file 是一个目录名，系统将在该目录之下建立一个或多个与 source_file 同名的链接文件，链接文件和被链接文件的内容完全相同。

如果 target_file 为一个文件，用户将被告知该文件已存在且不进行链接。
如果指定了多个 source_file 参数，那么最后一个target_file 参数必须为目录。
　
如果给 ln 命令加上 -s 选项，则建立符号链接。

如果 target_file 已经存在但不是目录，将不做链接。target_file 可以是任何一个文件名（可包含路径），也可以是一个目录，并且允许它与 source_file 不在同一个文件系统中。

如果 target_file 是一个已经存在的目录，系统将在该目录下建立一个或多个与 source_file 同名的文件，此新建的文件实际上是指向原 source_file 的符号链接文件。

    硬连接和复制的区别：
    几个硬连接＝几个名字的同一个房子，同一个源文件的硬连接，名字可以相同或不同，但地址（inode）是一样的， 所以硬连接被删除只是把相应的抹去，只有最后一个名字被抹去你才会找不到房子；而复制是建造一个一模一样的房子，当然地址（inode）就不同的了。
    
    硬链接和符号链接的区别：
    硬连接记录的是目标的 inode；符号链接相当于 windows 下的快捷方式。
    hard link 由于 inode 的缘故，只能在本分区中做 link；符号链接可以做跨分区的 link。
