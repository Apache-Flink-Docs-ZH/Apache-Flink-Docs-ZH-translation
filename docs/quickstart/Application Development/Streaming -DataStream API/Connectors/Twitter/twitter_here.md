<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<title>Document</title>
</head>
<!--
path: Project Setup/Linking with Optional Modules
here link: https://ci.apache.org/projects/flink/flink-docs-release-1.2/dev/linking.html
-->
<body>

<div>
	<p>
		项目设置/可选连接模块
		连接模块不包含在发布的二进制包中
		二进制发布,包含的jar包在lib文件夹里，这些lib中的jar会被自动添加到你发布的应用程序的类路径中。几乎所有的Flink类都在lib文件夹里,有少数例外，
		例如流连接器和一些最新添加的模块。
		要运行依赖这些模块的代码，你需要使他们在运行时可以访问，我们建议两个选项：
			1. 复制必需的jar文件到你所有的TaskManagers的lib文件夹里。注意,这样做了之后,你必须重新启动TaskManagers。
			2. 或者把需要的jar文件,与你的代码一起打包。
		后期的版本推荐考虑作为类加载器管理在Flink。
	</p>
</div>
	
<div>
	<p>
		使用Maven打包你的用户代码
		提供这些没有包含在Flink包中的依赖包,我们提供使用Maven的两个选择。
			1. Maven装配插件建立一个所谓的uber-jar（可执行jar）包含所有的依赖。装配配置是直线前进的，但由此生成的结果jar可能会变得庞大。
	           阅读 <a href="http://maven.apache.org/plugins/maven-assembly-plugin/usage.html">maven-assembly-plugin</a>  进一步了解更多信息。
			2. Maven 解压缩插件，解压的相关依赖的部分与你的代码。
			   使用后一种方法捆绑Kafka连接器，flink-connector-kafka 你将需要去添加添加类，从连接器和Kafka API 本身。将以下内容添加到插件部分。
	</p>
</div>

<div>
	<p>
		<span class="nc"><plugin></span>
		    <span class="nc"><groupId></span>org.apache.maven.plugins<span class="nc"></groupId></span>
		    <span class="nc"><artifactId>maven-dependency-plugin<span class="nc"></artifactId></span>
		    <span class="nc"><version>2.9</version></span>
		    <span class="nc"><executions></span>
		        <span class="nc"><execution></span>
		            <span class="nc"><id></span>unpack<span class="nc"></id></span>
		            <!-- executed just before the package phase -->
		            <span class="nc"><phase></span>prepare-package<span class="nc"></phase></span>
		            <span class="nc"><goals></span>
		                <span class="nc"><goal></span>unpack<span class="nc"></goal></span>
		            <span class="nc"></goals></span>
		            <span class="nc"><configuration></span>
		                <span class="nc"><artifactItems></span>
		                    <!-- For Flink connector classes -->
		                    <span class="nc"><artifactItem></span>
		                        <span class="nc"><groupId></span>org.apache.flink<span class="nc"></groupId></span>
		                        <span class="nc"><artifactId></span>flink-connector-kafka<span class="nc"></artifactId></span>
		                        <span class="nc"><version></span>1.2.0<span class="nc"></version></span>
		                        <span class="nc"><type></span>jar<span class="nc"></type></span>
		                        <span class="nc"><overWrite></span>false<span class="nc"></overWrite></span>
		                        <span class="nc"><outputDirectory></span>${project.build.directory}/classes<span class="nc"></outputDirectory></span>
		                        <span class="nc"><includes></span>org/apache/flink/**<span class="nc"></includes></span>
		                    <span class="nc"></artifactItem></span>
		                    <!-- For Kafka API classes -->
		                    <span class="nc"><artifactItem></span>
		                        <span class="nc"><groupId>org.apache.kafka<span class="nc"></groupId></span>
		                        <span class="nc"><artifactId>kafka_<YOUR_SCALA_VERSION><span class="nc"></artifactId></span>
		                        <span class="nc"><version><YOUR_KAFKA_VERSION><span class="nc"></version></span>
		                        <span class="nc"><type>jar<span class="nc"></type></span>
		                        <span class="nc"><overWrite>false<span class="nc"></overWrite></span>
		                        <span class="nc"><outputDirectory>${project.build.directory}/classes<span class="nc"></outputDirectory></span>
		                        <span class="nc"><includes></span>kafka/**<span class="nc"></includes></span>
		                    <span class="nc"></artifactItem></span>
		                <span class="nc"></artifactItems></span>
		            <span class="nc"></configuration></span>
		        <span class="nc"></execution></span>
		     <span class="nc"></executions></span>
		<span class="nc"></plugin></span>
	</p>
</div>
     
<div>现在，当运行 man clean package 就会生成包含需要的依赖的新jar。</div>     
   
</body>
</html>
