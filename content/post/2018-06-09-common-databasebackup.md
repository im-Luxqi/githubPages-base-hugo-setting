---
title: 数据库备份 
date: 2018-06-07
tags: ["基础杂记"]
---


<!--more-->

这两天项目有数据库备份的要求,花了一段时间整理一份通用的备份接口方便以后的查阅。<br/>

**关于数据库备份**	<br/>
1.对于postgre数据库:pg_dump的相关指令可以通过 pg_dump --help查看,其他数据库同理;<br/>
2.backupDb方法,方法体内通过调用外部进程方式备份数据库;
#### 一、配置文件
{{< highlight javascript>}}
# db backup
sys.backup.db.username=root
sys.backup.db.password=root
sys.backup.dir=Z:/backup/mysql
sys.backup.dbname=3f-uat
#sys.backup.db.keep=6
sys.backup.db.database=mysql
pg96.dir=D:/PostgreSQL/10/bin/pg_dump
pg96.cmd={0},-U{1},-d{2},-f{3},-Eutf8,-h127.0.0.1,-p5432
pg96.env=PGPASSWORD:root,
mysql.dir=C:/Program Files/MySQL/MySQL Server 5.6/bin/mysqldump
mysql.cmd={0},-u{1},-p{4},-r{3},--set-charset=UTF8,{2},-h127.0.0.1,
mysql.env=
#oracle.dir=
#oracle.cmd=
#oracle.env=
{{</ highlight >}}
#### 二、备份接口
{{< highlight javascript>}}
public static boolean backupDb() {
/*
* 从配置文件中解析所要参数
*   dbname： 数据库名称
*   username: 进入数据库所需要的用户名
*   password: 进入数据库所需要的口令
*   cmd: 指令
*   cmdDir: 指令路径
*   backupDir：备份文件夹路径
*   backuppath： 备份文件路径
*   keepNumber： 配置数据库备份文件保留个数
* */
String database = AppConfig.getConfig("sys.backup.db.database");
String dbname = AppConfig.getConfig("sys.backup.dbname");
String username = AppConfig.getConfig("sys.backup.db.username");
String password = AppConfig.getConfig("sys.backup.db.password");
String backupDir = AppConfig.getConfig("sys.backup.dir");
String keepNumber = AppConfig.getConfig("sys.backup.db.keep");
String cmd = AppConfig.getConfig(database+".cmd");
String cmdDir = AppConfig.getConfig(database+".dir");
// env环境(prostgre数据库无法直接从命令行输入密码,这里通过配置环境属性“PGPASSWORD”来解决)
Map<String, String> envMap = new HashMap<>();
String envstring = AppConfig.getConfig(database+".env");
String[] split = envstring.split(",");
for(String string : split) {
	String[] sp = string.split(":");
	if(sp.length == 2){
		envMap.put(sp[0],sp[1]);
	}
}
// 备份存放地址
Date d = new Date();
SimpleDateFormat sdf = new SimpleDateFormat("yyyyMMddHHmmss");
String filename =dbname + "-"  + sdf.format(d)+   ".tar"; // 备份文件名称

File saveFile = new File(backupDir);
if (!saveFile.exists()) { // 如果目录不存在
	saveFile.mkdirs();  // 创建文件夹
}
String backuppath = backupDir +"/"+ filename;


/*备份数据库*/
boolean flag = true;// 备份是否成功
Process p;
ProcessBuilder pb;
Runtime rt = Runtime.getRuntime();
String cmdTemp = MessageFormat.format(cmd,cmdDir,username,dbname,backuppath,password);// 命令模版
List<String> options = Arrays.asList(cmdTemp.split(","));
pb = new ProcessBuilder(options);
try {
	final Map<String, String> env = pb.environment();
	env.putAll(envMap); // 环境设置
	p = pb.start(); // 备份操作
	final BufferedReader r = new BufferedReader(
			new InputStreamReader(p.getErrorStream()));
	String line = r.readLine();
	while (line != null)

	{ System.err.println(line); line = r.readLine(); }
	r.close();
	p.waitFor();
	System.out.println(p.exitValue());

} catch (IOException | InterruptedException e) {
	System.out.println(e.getMessage());
	flag = false;
}

/*控制备份文件保留个数 (文件数超过限制，保留最新的)*/
if(flag && StringUtils.isNotBlank(keepNumber)) {
	keepFileNumber(backupDir,Integer.parseInt(keepNumber));
}

return flag;
}



/**
* 根据最大备份数，保留备份文件
* @param strPath
* @param keepNum
*/
public static void keepFileNumber(String strPath,int keepNum) {
/*寻找符合条件的文件*/
List<File> filelist = new ArrayList<File>();
File dir = new File(strPath);
File[] files = dir.listFiles(); // 该文件目录下文件全部放入数组
if (files != null) {
	for (int i = 0; i < files.length; i++) {
		String fileName = files[i].getName();
		if (!files[i].isDirectory() 
			&& fileName.endsWith("tar") 
			&& fileName.startsWith("rexframe")) {
			filelist.add(files[i]);
		}
	}
}

/*排序(按日期倒序排列)*/
Collections.sort(filelist, new Comparator<File>() {
	@Override
	public int compare(File o1, File o2) {
		return o2.getName().compareTo(o1.getName());
	}
});

/*只保留最大保留数*/
for (int i = 0; i < filelist.size(); i++) {
	if( i > keepNum-1) {
		filelist.get(i).delete();
	}
}
}
{{</ highlight >}}

#### 三、恢复数据库
{{< highlight javascript>}}

/**
 * 恢复（数据库恢复一般不会在当前程序中直接一键恢复）
 * 调整相应的命令模板
 * 例如prostgre数据库： 
 */
 
String cmd = D:/PostgreSQL/10/bin;
String cmdTemp = MessageFormat.format("{0}/psql,-U{1},-d{2},-f{3}",cmd,username,dbname,backuppath);

{{</ highlight >}}