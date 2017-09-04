---
title: "Program Packaging and Distributed Execution"
nav-title: Program Packaging
nav-parent_id: execution
nav-pos: 20
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->


����֮ǰ������������,Flink�ĳ��������`Զ�̻�����remote environment��`�ķ�ʽִ���ڼ�Ⱥ�ϡ�
ͬʱ������Ҳ����ѡ���ô��JAR����Java Archives���ķ�ʽ��ִ�С����������ʹ��[������command line interface]({{ site.baseurl }}/setup/cli.html)���������ǵ�ǰ��������

### �������Packaging Programs��

Ϊ��֧��JAR��ͨ�������л�������ӿڵķ�ʽִ�У������ڲ�����ʹ�ð���`StreamExecutionEnvironment.getExecutionEnvironment()`��
��������JAR��ͨ�������л�������ӿ��ύʱ�������������Ϊ��Ⱥִ�еĻ��������Flink����û��ͨ����Щ�ӿڵ��ã�
��ô����������Ϊһ�����ػ�����ִ�С�

Ϊ�˴�����������Լ򵥵Ľ����е��ർ����һ��JAR����JAR����manifestһ��Ҫָ������г���
*��ڵ㣨entry point��*���й���`main`���������ࡣ��򵥵�һ�ַ�ʽ���ǽ�*main-class*����manifest����������:`main-class: org.apache.flinkexample.MyProgram`�������*main-class* ������ָ������������Java�������ͨ��������`java -jar pathToTheJarFile`
ִ��JAR��ʱ�ҵ�����������һ��������������� IDE���ṩ���ڵ���JAR��ʱ�Զ���������԰���������Ĺ��ܡ�

### ͨ���ƻ����������Packaging Programs through Plans��

���⣬���ǻ�֧�ֽ���������*�ƻ���Plans)*�� �ƻ���������һ�����������������������*����ƻ���Program Plan��*��
���������������ж���һ���������ڻ����е���`execute()`ִ�к�����Ϊ��������һ���أ��������Ҫʵ��`org.apache.flink.api.common.Program`
�ӿڣ�����`getPlan(String...)`��������������е��ַ���������һ�������в������ڴ������ƻ��Ĺ����У�JAR����manifest����ָ��ʵ�ֵ�`org.apache.flinkapi.common.Program`�ӿڣ����������������ࡣ

### �ܽᣨSummary��

�ܵ���˵������һ������ĳ���������¼������裺

1. JAR����manifestѰ��*main-class* ���� *program-class*���ԡ�����������Զ��ҵ��ˣ���*program-class* Ҫ������*main-class* ���ԡ�
��JAR��manifest�ļ��������κ�һ������ʱ�������л�������ӿڶ�֧�ֽ�������ڵ��������ֶ�����ķ�ʽ��

2. ��������ʵ����`org.apache.flinkapi.common.Program`�ӿڣ���ôϵͳ������`getPlan(String...)`������ִ�в���ó���ƻ���

3. ��������û��ʵ��`org.apache.flinkapi.common.Program`�ӿڣ���ôϵͳ���������е���������

{% top %}
