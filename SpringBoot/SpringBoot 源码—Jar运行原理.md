#  SpringBoot 源码解析—— SpringBoot Jar运行原理

[TOC]

## 1.  概述

​	SpringBoot项目打包后生成的Jar包，我们可以通过命令 `java -jar` 启动运行。SpringBoot 提供 `spring-boot-maven-plugin` 插件，可以将SpringBoot 应用快速打包为 `jar` 或 `war` 包，那么，`jar`包是如何运行，并启动SpringBoot应用呢？本文一起来探索 Spring Boot Jar 启动运行原理。

## 2. `java -jar` 做了什么？

​	要想明白Spring Boot Jar 运行原理，我们先来看下启动命令 `java -jar` 都做了什么，看看 Oracle[官网介绍](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html)：  

> By default, the first argument that is not an option of the `java` command is the fully qualified name of the class to be called. If the `-jar` option is specified, its argument is the name of the JAR file containing class and resource files for the application. The startup class must be indicated by the `Main-Class` manifest header in its source code.

* `java` 命令第一个参数为需调用类的全限定类名
* `-jar` 参数设定时，参数名为 JAR 包名，其中包含 类文件和资源文件。
* `Main-Class` 为启动类，必须在Manifest 头中限定

所以，`java -jar` 运行时会查找 Manifest 文件中指定的`Mian-Class` 启动类。

## 3. Spring Boot  JAR 结构

​	打开任意一个SpringBoot JAR 包，主要结构包括：

```
├─BOOT-INF
│  ├─classes
│  └─lib
├─META-INF
│  │  MANIFEST.MF
│  │  
│  └─maven
│      └─top.xgoding
│          └─sms-spring-boot-starter-test
└─org
    └─springframework
        └─boot
            └─loader
                ├─archive
                ├─data
                ├─jar
                ├─jarmode
                └─util
```

各文件夹作用如下：

1. **META-INF** 目录，`MANIFEST.MF` 文件提供JAR包元数据，声明JAR启动类。
2. **org** 目录，为 `spring-boot-loader` 项目，为 `java -jar` 命令启动程序，同时为`SpringBoot`应用启动提供引导，解决JAR包嵌套类加载的问题。
3. **BOOT-INF/classes** 目录，为`SpringBoot`项目编写的类文件。
4. **BOOT-INF/lib** 目录，为`SpringBoot`项目依赖的JAR包。

`spring-boot-loader`作为应用的入口，需要解决两个问题：

1. 如何引导执行`SpringBoot`应用的启动类，如 `XXXSpringBootApplication`。
2. 如何加载`BOOT-INF/`中 classes 和 lib 到 `classpath`中进行执行。

​	接下来，我们先看下`MANIFEST.MF`文件的配置信息。

## 4.  MANIFEST.MF 元数据

​	打开`META-INF/MANIFEST.MF`文件，如下所示：

```
Manifest-Version: 1.0
Spring-Boot-Classpath-Index: BOOT-INF/classpath.idx
Implementation-Title: sms-spring-boot-starter-test
Implementation-Version: 1.0-SNAPSHOT
Spring-Boot-Layers-Index: BOOT-INF/layers.idx
Start-Class: top.xgoding.lab.starter.sms.test.SmsApplication
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Build-Jdk-Spec: 1.8
Spring-Boot-Version: 2.6.6
Created-By: Maven Jar Plugin 3.2.0
Main-Class: org.springframework.boot.loader.JarLauncher
```

此文件详细描述了JAR包的各项配置，每一行为一个配置项，其中重点配置项为：

- **Main-Class** : 指定JAR包的启动类，为`org.springframework.boot.loader.JarLauncher`，这里的类即为 `java -jar` 命令运行时需要启动的入口类，执行该类中的 `main` 方法。
- **Start-Class** : 指定 `SpringBoot`应用的启动类，为 `top.xgoding.lab.starter.sms.test.SmsApplication`，这里的类即为 `@SpringBootApplication` 标注的应用启动类。

> 注：**Start-Class** 相关配置信息，是由 `spring-boot-maven-plugin` 插件打包生成，使 `spring-boot-loader` 正确引导启动应用。

> 注：Java 中规定可执行 jar 包中不允许jar包嵌套，并且必须遵守jar的加载规则。所以这里我们无法直接通过 `java -jar` 命令直接启动 `Start-Class` 中的 `main`方法运行程序。`spring-boot-loader` 自定义实现了 `ClassLoader` 用于加载 `BOOT-INF/classes`、`BOOT-INF/lib` 类文件及Jar包，用于启动应用。

## 5. JarLauncher 启动流程

​	通过上面分析，我们了解到 `SpringBoot`项目是通过 `org.springframework.boot.loader.JarLauncher` 来启动应用，接下来我们先学习 `JarLauncher`相关知识，先看下类的继承关系：

![JarLauncher](http://image.xgoding.top/spring-boot/JarLauncher.png)   

`JarLauncher` 继承自 `ExecutableArchiveLauncher` 及`Launcher` 类，通过 `main`方法创建 `JarLauncher` 对象，并带入参数运行 `launcher` 方法，如下所示：

```
public class JarLauncher extends ExecutableArchiveLauncher {

……省略……

	public static void main(String[] args) throws Exception {
		new JarLauncher().launch(args);
	}

}
```

`launch()` 方法的具体实现由父类 `org.springframework.boot.loader.Launcher#launch(java.lang.String[])` 实现，如下所示：

```
	protected void launch(String[] args) throws Exception {
		// exploded mode : 将 jar 包解压后运行的方式
		// 如果非 exploded mode 需要注册URL handler 加载 jar包。
		if (!isExploded()) {
			//注册 URL 协议的处理器
			JarFile.registerUrlProtocolHandler();
		}
		// 创建ClassLoader
		ClassLoader classLoader = createClassLoader(getClassPathArchivesIterator());
		// jarmode：创建docker镜像时配置的layertools参数，暂时先不管
		String jarMode = System.getProperty("jarmode");
		// 如果无 jarmode 参数，通过 getMainClass() 方法 读取 Archive变量内容，获取 MATA-INF/MANIFEST.MF 文件中 Start-Class 属性值
		String launchClass = (jarMode != null && !jarMode.isEmpty()) ? JAR_MODE_LAUNCHER : getMainClass();
		// 运行SpringBootApplication启动类
		launch(args, launchClass, classLoader);
	}
```

1. 首先判断JAR包运行模式，如果非 `exploeded` 模式，则需要注册 `SpringBoot` 自定义实现的 `java.net.URLStreamHandler` 来加载 JAR 包。
2. 调用  `createClassLoader` 方法创建自定义` `ClassLoader` ，从 JAR 包中加载类。
3. 调用 `getMainClass` 方法获取 SpringBoot 应用中指定的启动类，即 `Start-Class` 中配置的类。
4. 调用 `launch` 方法执行 SpringBoot 应用中声明的启动类，启动应用程序。

下面我们来详细分析执行过程。

### 5.1 注册 `URLStreamHandler` ( registerUrlProtocolHandler )

`org.springframework.boot.loader.Launcher#launch(java.lang.String[])` 方法中第一步，注册 `URLStreamHandler`，如下：

```
		// exploded mode : 将 jar 包解压后运行的方式
		// 如果非 exploded mode 需要注册URL handler 加载 jar包。
		if (!isExploded()) {
			//注册 URL 协议的处理器
			JarFile.registerUrlProtocolHandler();
		}
```

> 注：在 `org.springframework.boot.loader.Launcher#launch(java.lang.String[])` 中使用的 `JarFile` 为  `java.util.jar.JarFile` 的子类，主要用于读取JAR文件内容。

`org.springframework.boot.loader.jar.JarFile#registerUrlProtocolHandler` 此方法用于注册 `SpringBoot` 自定义的URL协议处理器，详细过程如下：

```
/**
	 * Register a {@literal 'java.protocol.handler.pkgs'} property so that a
	 * {@link URLStreamHandler} will be located to deal with jar URLs.
	 */
	public static void registerUrlProtocolHandler() {
		// 配置Jar上下文URL，用于后续回调
		Handler.captureJarContextUrl();
		// 获取UrlStreamHandler路径
		String handlers = System.getProperty(PROTOCOL_HANDLER, "");
		// 增加 SpringBoot 自定义的 HANDLERS_PACKAGE (org.springframework.boot.loader)
		System.setProperty(PROTOCOL_HANDLER,
				((handlers == null || handlers.isEmpty()) ? HANDLERS_PACKAGE : handlers + "|" + HANDLERS_PACKAGE));
		// 重置已缓存的 URLStreamHandler
		resetCachedUrlHandlers();
	}
```

通过进行上述操作，将 `HANDLERS_PACKAGE` `(org.springframework.boot.loader)` 包设置到 `PROTOCOL_HANDLER` `(java.protocol.handler.pkgs)` 环境变量中，从而使用自定义`URLStreamHandler` 实现类 `org.springframework.boot.loader.jar.Handler` 处理`jar://`协议的URL。

> ​	注：`spring-boot-loader`项目提供 `org.springframework.boot.loader.jar.HandlerTests` 测试类，可对Handler进行测试。

> ​	注：关于 Handler 实现原理，可参考 [Java URL 协议扩展实现]()

### 5.2  自定义类加载 （createClassLoader）

​	`org.springframework.boot.loader.Launcher#launch(java.lang.String[])` 方法中第二步，自定义类加载器，如下：

```
		// 创建ClassLoader
		ClassLoader classLoader = createClassLoader(getClassPathArchivesIterator());
```

#### 5.2.1. 构造`classpath` 文档结构 （getClassPathArchivesIterator）

​	返回所有符合条件的 jar 或 工程文件，并包装成一个类型为 `Archive` 的 Iterator 对象。这里的符合条件是指在 `BOOT-INF/lib/` 下的 jar 文件，和在 `BOOT-INF/classes/` 下的所有工程文件，用来构建 classpath，`org.springframework.boot.loader.Launcher` 中代码如下：

```
	/**
	 * Returns the archives that will be used to construct the class path.
	 * @return the class path archives
	 * @throws Exception if the class path archives cannot be obtained
	 * @since 2.3.0
	 */
	protected Iterator<Archive> getClassPathArchivesIterator() throws Exception {
		return getClassPathArchives().iterator();
	}
	
```

具体实现类`org.springframework.boot.loader.ExecutableArchiveLauncher`中，代码实现如下：

```
	@Override
	protected Iterator<Archive> getClassPathArchivesIterator() throws Exception {
		// <1> 创建过滤条件，是否需要进一步搜索
		Archive.EntryFilter searchFilter = this::isSearchCandidate;
		// <2> 获取所有Archive
		Iterator<Archive> archives = this.archive.getNestedArchives(searchFilter,
				(entry) -> isNestedArchive(entry) && !isEntryIndexed(entry));
		if (isPostProcessingClassPathArchives()) {
			// <3> 后续处理
			archives = applyClassPathArchivePostProcessing(archives);
		}
		return archives;
	}
```

1.  创建过滤条件，判断是否需要进一步搜索，检查文件路径是否以 `BOOT-INF/` 开始，代码如下：

`org.springframework.boot.loader.ExecutableArchiveLauncher#isSearchCandidate` 实现如下：

```
	protected boolean isSearchCandidate(Archive.Entry entry) {
		// 如果前缀不存在，需要进一步搜索
		if (getArchiveEntryPathPrefix() == null) {
			return true;
		}
		// Entry是否以指定前缀开头
		return entry.getName().startsWith(getArchiveEntryPathPrefix());
	}
```

`org.springframework.boot.loader.JarLauncher#getArchiveEntryPathPrefix` 子类实现如下：

```
	@Override
	protected String getArchiveEntryPathPrefix() {
		return "BOOT-INF/";
	}
```



2. 获取所有 `Archive` 对象。根据传入 `org.springframework.boot.loader.archive.Archive.EntryFilter`  的 `SearchFilter` 和 `includeFilter`  相结合进行过滤后，返回 `Archive`对象集合。

`org.springframework.boot.loader.archive.Archive#getNestedArchives(org.springframework.boot.loader.archive.Archive.EntryFilter, org.springframework.boot.loader.archive.Archive.EntryFilter)` 默认实现如下：

```
	default Iterator<Archive> getNestedArchives(EntryFilter searchFilter, EntryFilter includeFilter)
			throws IOException {
		//searchFilter 和 includeFilter 需要同时满足
		EntryFilter combinedFilter = (entry) -> (searchFilter == null || searchFilter.matches(entry))
				&& (includeFilter == null || includeFilter.matches(entry));
		List<Archive> nestedArchives = getNestedArchives(combinedFilter);
		return nestedArchives.iterator();
	}
```

`org.springframework.boot.loader.archive.JarFileArchive#getNestedArchives` 实际实现如下：

```
	@Override
	public Iterator<Archive> getNestedArchives(EntryFilter searchFilter, EntryFilter includeFilter) throws IOException {
		return new NestedArchiveIterator(this.jarFile.iterator(), searchFilter, includeFilter);
	}
```

`org.springframework.boot.loader.archive.JarFileArchive.NestedArchiveIterator` 是 `org.springframework.boot.loader.archive.JarFileArchive.AbstractIterator` 静态内部类的子类，同时实现了 `java.util.Iterator` 接口，通过实现父类抽象方法`org.springframework.boot.loader.archive.JarFileArchive.AbstractIterator#adapt` ，为每个Entry 提供获取 `Archive` 对象的方法 `getNestedArchive(Entry entry)`。

```
	@Override
	public Iterator<Archive> getNestedArchives(EntryFilter searchFilter, EntryFilter includeFilter) throws IOException {
		return new NestedArchiveIterator(this.jarFile.iterator(), searchFilter, includeFilter);
	}
	
	……省略……
	
	private class NestedArchiveIterator extends AbstractIterator<Archive> {

		NestedArchiveIterator(Iterator<JarEntry> iterator, EntryFilter searchFilter, EntryFilter includeFilter) {
			super(iterator, searchFilter, includeFilter);
		}

		@Override
		protected Archive adapt(Entry entry) {
			try {
				// 获取每个条目的Archive对象
				return getNestedArchive(entry);
			}
			catch (IOException ex) {
				throw new IllegalStateException(ex);
			}
		}

	}
	
	……省略……
	
	protected Archive getNestedArchive(Entry entry) throws IOException {
		JarEntry jarEntry = ((JarFileEntry) entry).getJarEntry();
		if (jarEntry.getComment().startsWith(UNPACK_MARKER)) {
			return getUnpackedNestedArchive(jarEntry);
		}
		try {
			JarFile jarFile = this.jarFile.getNestedJarFile(jarEntry);
			return new JarFileArchive(jarFile);
		}
		catch (Exception ex) {
			throw new IllegalStateException("Failed to get nested archive for entry " + entry.getName(), ex);
		}
	}
```

同时，静态抽象类 `org.springframework.boot.loader.archive.JarFileArchive.AbstractIterator` 中的 `poll()` 方法，通过 `searchFilter` 和 `includeFilter` 判断 Entry 是否满足 `Spring Boot`需加载的类及Jar文件。代码如下：

```
private abstract static class AbstractIterator<T> implements Iterator<T> {

		……省略……

		@Override
		public T next() {
			T result = adapt(this.current);
			this.current = poll();
			return result;
		}
		
		private Entry poll() {
			while (this.iterator.hasNext()) {
				JarFileEntry candidate = new JarFileEntry(this.iterator.next());
				if ((this.searchFilter == null || this.searchFilter.matches(candidate))
						&& (this.includeFilter == null || this.includeFilter.matches(candidate))) {
					return candidate;
				}
			}
			return null;
		}

		protected abstract T adapt(Entry entry);

	}
```

其中，**`searchFlter`** 由上述第一步<1.>中 `org.springframework.boot.loader.ExecutableArchiveLauncher#isSearchCandidate` 方法实现，如下：

```
		// <1> 创建过滤条件，是否需要进一步搜索
		Archive.EntryFilter searchFilter = this::isSearchCandidate;
```

**`includeFilter`** 由上述第二步<2>中 `org.springframework.boot.loader.JarLauncher#isNestedArchive` 和 `org.springframework.boot.loader.ExecutableArchiveLauncher#isEntryIndexed` 方法共同实现。

`org.springframework.boot.loader.JarLauncher#isNestedArchive` 方法，用于判断指定的条目是否应该加载在`classpath`中，代码如下：

```
	@Override
	protected boolean isNestedArchive(Archive.Entry entry) {
		return NESTED_ARCHIVE_ENTRY_FILTER.matches(entry);
	}
```

其中 `NESTED_ARCHIVE_ENTRY_FILTER` EntryFilter 对象，用于查找的 match 方法如下：

```
	static final EntryFilter NESTED_ARCHIVE_ENTRY_FILTER = (entry) -> {
		if (entry.isDirectory()) {
			return entry.getName().equals("BOOT-INF/classes/");
		}
		return entry.getName().startsWith("BOOT-INF/lib/");
	};
```

此处，可看出 `spring-boot-loader` 构造的 `classpath` 只包括 `BOOT-INF/classes/`目录中的类文件 和 `BOOT-INF/lib/` 目录中的 JAR 包。

同时，`org.springframework.boot.loader.ExecutableArchiveLauncher#isEntryIndexed` 用于判断该条目在当前 `classPathIndex`对象中是否已索引，这里需要注意仅当 `Archive` 为 `org.springframework.boot.loader.archive.ExplodedArchive` 时才生效，`org.springframework.boot.loader.JarLauncher` 中 `classPathIndex` 属性为 null，因此该方法无效，实现如下：

```
	private boolean isEntryIndexed(Archive.Entry entry) {
		if (this.classPathIndex != null) {
			return this.classPathIndex.containsEntry(entry.getName());
		}
		return false;
	}
```

3. 对`Archive` 文件进行后续处理（post processing），实现如下：

```
	if (isPostProcessingClassPathArchives()) {
			// <3> 后续处理
			archives = applyClassPathArchivePostProcessing(archives);
	}
```

`org.springframework.boot.loader.ExecutableArchiveLauncher#isPostProcessingClassPathArchives` 方法用于判断是否需要对 `Archive` 方法进行后续处理，该方法默认实现返回`true`，实现如下：

```
	protected boolean isPostProcessingClassPathArchives() {
		return true;
	}
```

考虑到后向兼容性，如果子类在未复写父类`org.springframework.boot.loader.ExecutableArchiveLauncher#postProcessClassPathArchives`方法时，应返回`false`，比如 `org.springframework.boot.loader.JarLauncher#isPostProcessingClassPathArchives` 则直接返回 `false`:

```
	@Override
	protected boolean isPostProcessingClassPathArchives() {
		return false;
	}
```

`org.springframework.boot.loader.ExecutableArchiveLauncher#applyClassPathArchivePostProcessing` 方法用于构建对 `Archive` 的`post processing`处理操作，这里调用 `org.springframework.boot.loader.ExecutableArchiveLauncher#postProcessClassPathArchives` 方法实现，其实就是预留一个回调操作，在使用 `Archive`集合之前先进行处理， 此处实现方法为空：

```
	private Iterator<Archive> applyClassPathArchivePostProcessing(Iterator<Archive> archives) throws Exception {
		List<Archive> list = new ArrayList<>();
		while (archives.hasNext()) {
			list.add(archives.next());
		}
		postProcessClassPathArchives(list);
		return list.iterator();
	}
	
	protected void postProcessClassPathArchives(List<Archive> archives) throws Exception {
	}
```

> 小结：`org.springframework.boot.loader.Launcher#getClassPathArchivesIterator` 用于构建 `spring boot `jar 文件归档对象`Archive`集合，加载 `BOOT-INF/classes/` 和 `BOOT-INF/lib/` 目录下类文件和JAR文件。

#### 5.2.2  `Archive` 档案类

​	在上述构造文档结构时，我们一直使用了一个接口：`org.springframework.boot.loader.archive.Archive`，该接口用于抽象文档，其类图如下：

![Archive](http://image.xgoding.top/spring-boot/Archive.png)  

* `JarFileArchive` 用于对jar包的档案实现
* `ExplodedArchive` 用于对目录的档案实现

其中，`Archive` 中包含两个内部接口`EntryFilter` 和 `Entry` ，分别用于过滤及对文档条目的描述，其实现如下：

```
	/**
	 * Represents a single entry in the archive.
	 */
	interface Entry {

		/**
		 * Returns {@code true} if the entry represents a directory.
		 * @return if the entry is a directory
		 */
		boolean isDirectory();

		/**
		 * Returns the name of the entry.
		 * @return the name of the entry
		 */
		String getName();

	}

	/**
	 * Strategy interface to filter {@link Entry Entries}.
	 */
	@FunctionalInterface
	interface EntryFilter {

		/**
		 * Apply the jar entry filter.
		 * @param entry the entry to filter
		 * @return {@code true} if the filter matches
		 */
		boolean matches(Entry entry);

	}
```

那么，我们在 `ExecutableArchiveLauncher` 中执行`getClassPathArchivesIterator`等相关方法时，`this.archive` 对象是如何赋值的呢？我们查看类构造方法，这里有两个构造方法，此处`new JarLauncher().launch(args);`最终调用的是无参构造方法：

```
	public ExecutableArchiveLauncher() {
		try {
			this.archive = createArchive();
			this.classPathIndex = getClassPathIndex(this.archive);
		}
		catch (Exception ex) {
			throw new IllegalStateException(ex);
		}
	}

	protected ExecutableArchiveLauncher(Archive archive) {
		try {
			this.archive = archive;
			this.classPathIndex = getClassPathIndex(this.archive);
		}
		catch (Exception ex) {
			throw new IllegalStateException(ex);
		}
	}
```

可以看出`archive`对象由 `createArchive`方法创建，最终返回`root`档案对象，代码如下：

```
	protected final Archive createArchive() throws Exception {
		// 获取jar所在根文件的绝对路径
		ProtectionDomain protectionDomain = getClass().getProtectionDomain();
		CodeSource codeSource = protectionDomain.getCodeSource();
		URI location = (codeSource != null) ? codeSource.getLocation().toURI() : null;
		String path = (location != null) ? location.getSchemeSpecificPart() : null;
		if (path == null) {
			throw new IllegalStateException("Unable to determine code source archive");
		}
		File root = new File(path);
		if (!root.exists()) {
			throw new IllegalStateException("Unable to determine code source archive from " + root);
		}
		// 根文件为目录，创建 ExplodedArchive
		// 根文件不为目录，创建 JarFileArchive
		return (root.isDirectory() ? new ExplodedArchive(root) : new JarFileArchive(root));
	}
```

`root` 文件则为`JAR` 所在的绝对路径，最终会将`BOOT-INF/classes/`目录归档为一个`Archive`对象，`BOOT-INF/lib/`下的每一个内嵌 `JAR` 文件都会被归档为一个 `Archive` 对象。

#### 5.2.3 创建类加载器（createClassLoader）

在创建完成 `Archive`对象集合后，通过`org.springframework.boot.loader.ExecutableArchiveLauncher#createClassLoader` 加载这些归档文件。这里就需要自定义`ClassLoader`来加载这些文件，代码如下：

```
	@Override
	protected ClassLoader createClassLoader(Iterator<Archive> archives) throws Exception {
		// 获取所有URL地址
		List<URL> urls = new ArrayList<>(guessClassPathSize());
		while (archives.hasNext()) {
			urls.add(archives.next().getUrl());
		}
		if (this.classPathIndex != null) {
			urls.addAll(this.classPathIndex.getUrls());
		}
		// 创建加载这些URL的类加载器
		return createClassLoader(urls.toArray(new URL[0]));
	}
```

通过`org.springframework.boot.loader.Launcher#createClassLoader(java.net.URL[])`创建加载这些的URL的类加载器，返回自定义`LaunchedURLClassLoader` :

```
	protected ClassLoader createClassLoader(URL[] urls) throws Exception {
		return new LaunchedURLClassLoader(isExploded(), getArchive(), urls, getClass().getClassLoader());
	}
```

#### 5.2.4 自定义类加载器 （ LaunchedURLClassLoader）

`LaunchedURLClassLoader` 是 `spring-boot-loader` 自定义的类加载器，实现对 jar 包中`BOOT-INF/classes/`中的类和`BOOT-INF/lib/`下的 `JAR` 进行加载。

- [ ] 待补充

### 5.3   启动

#### 5.3.1 获取启动类（launchClass）

这里的`launchClass`是通过 `getMainClass()`方法获取，这里不展开对`jarMode`进行讨论，如下：

```
String launchClass = (jarMode != null && !jarMode.isEmpty()) ? JAR_MODE_LAUNCHER : getMainClass();
```

`getMainClass()` 是 `Launcher`类中定义的抽象函数，由`org.springframework.boot.loader.ExecutableArchiveLauncher#getMainClass` 负责实现：

```
	@Override
	protected String getMainClass() throws Exception {
		Manifest manifest = this.archive.getManifest();
		String mainClass = null;
		if (manifest != null) {
			mainClass = manifest.getMainAttributes().getValue(START_CLASS_ATTRIBUTE);
		}
		if (mainClass == null) {
			throw new IllegalStateException("No 'Start-Class' manifest entry specified in " + this);
		}
		return mainClass;
	}
```

获取`MANIFEST.MF`文件中定义的`Start-Class`全类名，即 `Spring boot `应用主启动类。

#### 5.3.2 执行启动

通过`launch(args, launchClass, classLoader);启动`launchClass`, 由 `org.springframework.boot.loader.Launcher#launch(java.lang.String[], java.lang.String, java.lang.ClassLoader)`负责实现：

```
	protected void launch(String[] args, String launchClass, ClassLoader classLoader) throws Exception {
		// 设置当前线程类加载器为 LaunchedURLClassLoader
		Thread.currentThread().setContextClassLoader(classLoader);
		// 创建 MainMethodRunner 对象，并执行 run() 方法启动应用
		createMainMethodRunner(launchClass, args, classLoader).run();
	}
```

`org.springframework.boot.loader.MainMethodRunner` 类主要通过反射机制获取`main`方法，启动SpringBoot 应用。

```
public class MainMethodRunner {

	private final String mainClassName;

	private final String[] args;

	/**
	 * Create a new {@link MainMethodRunner} instance.
	 * @param mainClass the main class
	 * @param args incoming arguments
	 */
	public MainMethodRunner(String mainClass, String[] args) {
		// Spring boot 应用主启动类
		this.mainClassName = mainClass;
		// 启动参数
		this.args = (args != null) ? args.clone() : null;
	}

	public void run() throws Exception {
		// 获取启动类对象，通过当前线程类加载器（LaunchedURLClassLoader） 保证加载到主类
		Class<?> mainClass = Class.forName(this.mainClassName, false, Thread.currentThread().getContextClassLoader());
		// 通过反射机制获取启动类Main方法
		Method mainMethod = mainClass.getDeclaredMethod("main", String[].class);
		mainMethod.setAccessible(true);
		// 执行Main方法，启动SpringBoot 应用
		mainMethod.invoke(null, new Object[] { this.args });
	}

}
```

## 6. 总结

- [ ] 待完善























