
	GIT - the stupid content tracker

"git" can mean anything, depending on your mood.

 - random three-letter combination that is pronounceable, and not
   actually used by any common UNIX command.  The fact that it is a
   mispronounciation of "get" may or may not be relevant.
 - stupid. contemptible and despicable. simple. Take your pick from the
   dictionary of slang.
 - "global information tracker": you're in a good mood, and it actually
   works for you. Angels sing, and a light suddenly fills the room. 
 - "goddamn idiotic truckload of sh*t": when it breaks

This is a stupid (but extremely fast) directory content manager.  It
doesn't do a whole lot, but what it _does_ do is track directory
contents efficiently. 

There are two object abstractions: the "object database", and the
"current directory cache".

	The Object Database (SHA1_FILE_DIRECTORY)

The object database is literally just a content-addressable collection
of objects.  All objects are named by their content, which is
approximated by the SHA1 hash of the object itself.  Objects may refer
to other objects (by referencing their SHA1 hash), and so you can build
up a hierarchy of objects. 

There are several kinds of objects in the content-addressable collection
database.  They are all in deflated with zlib, and start off with a tag
of their type, and size information about the data.  The SHA1 hash is
always the hash of the _compressed_ object, not the original one.

In particular, the consistency of an object can always be tested
independently of the contents or the type of the object: all objects can
be validated by verifying that (a) their hashes match the content of the
file and (b) the object successfully inflates to a stream of bytes that
forms a sequence of <ascii tag without space> + <space> + <ascii decimal
size> + <byte\0> + <binary object data>. 

BLOB: A "blob" object is nothing but a binary blob of data, and doesn't
refer to anything else.  There is no signature or any other verification
of the data, so while the object is consistent (it _is_ indexed by its
sha1 hash, so the data itself is certainly correct), it has absolutely
no other attributes.  No name associations, no permissions.  It is
purely a blob of data (ie normally "file contents"). 

TREE: The next hierarchical object type is the "tree" object.  A tree
object is a list of permission/name/blob data, sorted by name.  In other
words the tree object is uniquely determined by the set contents, and so
two separate but identical trees will always share the exact same
object. 

Again, a "tree" object is just a pure data abstraction: it has no
history, no signatures, no verification of validity, except that the
contents are again protected by the hash itself.  So you can trust the
contents of a tree, the same way you can trust the contents of a blob,
but you don't know where those contents _came_ from. 

Side note on trees: since a "tree" object is a sorted list of
"filename+content", you can create a diff between two trees without
actually having to unpack two trees.  Just ignore all common parts, and
your diff will look right.  In other words, you can effectively (and
efficiently) tell the difference between any two random trees by O(n)
where "n" is the size of the difference, rather than the size of the
tree. 

Side note 2 on trees: since the name of a "blob" depends entirely and
exclusively on its contents (ie there are no names or permissions
involved), you can see trivial renames or permission changes by noticing
that the blob stayed the same.  However, renames with data changes need
a smarter "diff" implementation. 

CHANGESET: The "changeset" object is an object that introduces the
notion of history into the picture.  In contrast to the other objects,
it doesn't just describe the physical state of a tree, it describes how
we got there, and why. 

A "changeset" is defined by the tree-object that it results in, the
parent changesets (zero, one or more) that led up to that point, and a
comment on what happened. Again, a changeset is not trusted per se:
the contents are well-defined and "safe" due to the cryptographically
strong signatures at all levels, but there is no reason to believe that
the tree is "good" or that the merge information makes sense. The
parents do not have to actually have any relationship with the result,
for example.

Note on changesets: unlike real SCM's, changesets do not contain rename
information or file mode chane information.  All of that is implicit in
the trees involved (the result tree, and the result trees of the
parents), and describing that makes no sense in this idiotic file
manager.

TRUST: The notion of "trust" is really outside the scope of "git", but
it's worth noting a few things. First off, since everything is hashed
with SHA1, you _can_ trust that an object is intact and has not been
messed with by external sources. So the name of an object uniquely
identifies a known state - just not a state that you may want to trust.

Furthermore, since the SHA1 signature of a changeset refers to the
SHA1 signatures of the tree it is associated with and the signatures
of the parent, a single named changeset specifies uniquely a whole
set of history, with full contents. You can't later fake any step of
the way once you have the name of a changeset.

So to introduce some real trust in the system, the only thing you need
to do is to digitally sign just _one_ special note, which includes the
name of a top-level changeset.  Your digital signature shows others that
you trust that changeset, and the immutability of the history of
changesets tells others that they can trust the whole history.

In other words, you can easily validate a whole archive by just sending
out a single email that tells the people the name (SHA1 hash) of the top
changeset, and digitally sign that email using something like GPG/PGP.

In particular, you can also have a separate archive of "trust points" or
tags, which document your (and other peoples) trust.  You may, of
course, archive these "certificates of trust" using "git" itself, but
it's not something "git" does for you. 

Another way of saying the same thing: "git" itself only handles content
integrity, the trust has to come from outside. 

	Current Directory Cache (".dircache/index")

The "current directory cache" is a simple binary file, which contains an
efficient representation of a virtual directory content at some random
time.  It does so by a simple array that associates a set of names,
dates, permissions and content (aka "blob") objects together.  The cache
is always kept ordered by name, and names are unique at any point in
time, but the cache has no long-term meaning, and can be partially
updated at any time. 

In particular, the "current directory cache" certainly does not need to
be consistent with the current directory contents, but it has two very
important attributes:

 (a) it can re-generate the full state it caches (not just the directory
     structure: through the "blob" object it can regenerate the data too)

     As a special case, there is a clear and unambiguous one-way mapping
     from a current directory cache to a "tree object", which can be
     efficiently created from just the current directory cache without
     actually looking at any other data.  So a directory cache at any
     one time uniquely specifies one and only one "tree" object (but
     has additional data to make it easy to match up that tree object
     with what has happened in the directory)


and

 (b) it has efficient methods for finding inconsistencies between that
     cached state ("tree object waiting to be instantiated") and the
     current state. 

Those are the two ONLY things that the directory cache does.  It's a
cache, and the normal operation is to re-generate it completely from a
known tree object, or update/compare it with a live tree that is being
developed.  If you blow the directory cache away entirely, you haven't
lost any information as long as you have the name of the tree that it
described. 

(But directory caches can also have real information in them: in
particular, they can have the representation of an intermediate tree
that has not yet been instantiated.  So they do have meaning and usage
outside of caching - in one sense you can think of the current directory
cache as being the "work in progress" towards a tree commit).

**中文翻译：**

GIT-愚蠢的内容跟踪器

根据您的心情，“ git”可以指任何东西。

- 明显的三字母组合，但不是实际上由任何常见的UNIX命令使用。这是一个事实“ get”的错误发音可能相关，也可能无关。
- 笨鄙视和卑鄙的。简单。从中选择语字典。
- “全球信息追踪器”：你心情很好，实际上为您工作。天使唱歌，一盏灯突然充满整个房间。
- “ sh * t的该死的愚蠢的卡车”：当它破碎时

这是一个愚蠢的（但速度非常快）目录内容管理器。它不会做很多事情，但是_does_做的是跟踪目录内容有效。

有两种对象抽象：“对象数据库”和“对象数据库”。
“当前目录缓存”。

**对象数据库（SHA1_FILE_DIRECTORY）**

对象数据库实际上只是一个内容可寻址的集合对象。所有对象均按其内容命名，即
用对象本身的SHA1哈希值近似。对象可以引用到其他对象（通过引用其SHA1哈希），因此您可以构建建立对象的层次结构。

内容可寻址集合中有几种对象数据库。它们都使用zlib压缩，并以标签开始类型，有关数据的大小信息。 SHA1哈希为始终是_compressed_对象的哈希，而不是原始对象的哈希。

特别是，始终可以测试对象的一致性与对象的内容或类型无关：所有对象都可以通过验证（a）它们的哈希值是否与文件，并且（b）对象成功膨胀为一个字节流，该字节流形成`<ascii tag without space> + <space> + <ascii decimal size> + <byte\0> + <binary object data>`。

**BLOB**：“ blob”对象不过是二进制数据blob，并且不是引用其他任何东西。没有签名或任何其他验证数据，因此虽然对象是一致的（它是_is_由其索引sha1哈希，因此数据本身当然是正确的），它具有绝对没有其他属性。没有名称关联，没有权限。它是纯粹是一团数据（即通常为“文件内容”）。

**树**：下一个分层对象类型是“树”对象。一颗树object是权限/名称/ blob数据的列表，按名称排序。其他单词树对象由设置内容唯一地确定，依此类推两棵独立但相同的树将始终共享完全相同的树目的。

同样，“树”对象只是纯数据抽象：它没有历史记录，没有签名，没有有效性验证，除了内容再次受到哈希本身的保护。所以你可以相信树的内容，就像您信任Blob的内容一样，但您不知道这些内容从何而来。

关于树的旁注：由于“树”对象是的排序列表“文件名+内容”，您可以在两棵树之间创建差异，而无需实际上不得不拆开两棵树。只需忽略所有公共部分，然后您的差异看起来正确。换句话说，您可以有效地（和有效地）用O（n）分辨任意两棵随机树之间的差异其中“ n”是差异的大小，而不是树。

关于树的旁注2：由于“斑点”的名称完全取决于专门针对其内容（即没有名称或权限请注意，您可以看到一些琐碎的重命名或权限更改斑点保持不变。但是，重命名需要更改数据更智能的“差异”实现。

**CHANGESET**：“ changeset”对象是引入将历史的概念融入到图片中。与其他物体相比，它不仅描述了树的物理状态，还描述了如何使我们到达那里，为什么。

“变更集”由其产生的树对象定义，导致这一点的父变更集（零，一个或多个），以及评论发生了什么。同样，变更集本身是不可信的：由于密码的原因，内容明确且“安全”各个级别都有强大的签名，但是没有理由相信树是“好”或合并信息有意义。的父母实际上不必与结果有任何关系，例如。

关于变更集的注意事项：与真实的SCM不同，变更集不包含重命名信息或文件模式链信息。所有这些都隐含在涉及的树（结果树，以及父母），并且在这个愚蠢的文件中进行描述是没有意义的经理。

信任：“信任”的概念确实超出了“ git”的范围，但是值得注意的是几件事。首先，因为所有内容都经过哈希处理
使用SHA1，您可以__ _相信对象是完整的，并且尚未被外部资源所困扰。因此，对象的名称唯一标识已知状态-而不是您可能希望信任的状态。

此外，因为变更集的SHA1签名是指与之关联的树的SHA1签名和签名的名称，单个命名的变更集唯一地指定了整个
一整套历史，内容完整。您以后无法伪造任何拥有变更集名称后的方式。

因此，要引入对系统的真正信任，您唯一需要的就是要做的只是对_one_特别说明进行数字签名，其中包括顶级变更集的名称。您的数字签名向其他人表明您相信变革，以及历史的不变性变更集告诉其他人，他们可以信任整个历史。

换句话说，您只需发送即可轻松验证整个存档发出一封电子邮件，告诉人们顶部的名称（SHA1哈希）变更集，并使用GPG / PGP之类的方式对该电子邮件进行数字签名。

特别是，您还可以拥有一个单独的“信任点”档案，或者标签，用于记录您（和其他人）的信任。您可能当然，请使用“ git”本身将这些“信任证书”存档，但是这不是“ git”为您做的事情。

说同一件事的另一种方式：“ git”本身仅处理内容完整性，信任必须来自外部。

    当前目录缓存（“ .dircache / index”）

“当前目录缓存”是一个简单的二进制文件，其中包含一个随机有效地表示虚拟目录内容
时间。它是通过一个将一组名称相关联的简单数组来实现的，日期，权限和内容（又称“ blob”）对象。缓存始终按名称排序，并且名称在任何时候都是唯一的时间，但缓存没有长期意义，可以部分随时更新。

特别是，“当前目录缓存”当然不需要与当前目录内容一致，但是它有两个非常重要属性：

 （a）**它可以重新生成其缓存的完整状态**（而不仅仅是目录结构：通过“ blob”对象，它也可以重新生成数据）

    作为一种特殊情况，存在清晰明确的单向映射从当前目录缓存到“树对象”，
    可以是仅从当前目录缓存有效创建，而无需实际上正在查看其他任何数据。
    所以目录缓存在任何一次唯一地指定一个并且只有一个“树”对象
    （但是有其他数据可以轻松匹配该树对象以及目录中发生的事情）
 和
 （b）**有效的方法来找出两者之间的不一致之处缓存状态**（“等待实例化的树对象”）和当前状态。

这是目录缓存仅做的两件事。它是缓存，正常的操作是完全从已知树对象，或将其与正在使用的活动树进行更新/比较发达。如果您完全删除了目录缓存，则没有只要您知道树的名称，它就会丢失任何信息描述。

（但是目录缓存中也可以包含真实信息：特别地，它们可以具有中间树的表示尚未实例化。因此它们确实具有意义和用法缓存之外-从某种意义上讲，您可以想到当前目录缓存为对树提交的“进行中的工作”）。
