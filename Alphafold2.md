<center><h1>超算9期上安装Alphafold 2</h1></center>

由于9期上没有提供Docker，这里依照[Xu Kui](https://github.com/kuixu/alphafold "https://github.com/kuixu/alphafold")提供的安装步骤，稍加修改。
<h3>1. 配置CUDA环境</h3>

CUDA版本不能低于11.1。9期上已经安装好CUDA 11.4，只需在.bashrc里配置即可。

<pre><code>echo '#cuda11.4' >>~/.bashrc
echo 'export PATH=/data/apps/cuda-11.4/bin:$PATH' >>~/.bashrc
echo 'export LD_LIBRARY_PATH=/data/apps/cuda-11.4/lib64:$LD_LIBRARY_PATH' >>~/.bashrc
</code></pre>

<h3>2. 下载并安装anaconda</h3>

<pre><code>wget https://repo.anaconda.com/archive/Anaconda3-2021.11-Linux-x86_64.sh
chmod +x Anaconda3-2021.11-Linux-x86_64.sh
./Anaconda3-2021.11-Linux-x86_64.sh
</code></pre>

安装过程中选择安装目录，安装程序会自动配置bash环境。如果要立即配置anaconda，可以运行

<pre><code>source ~/.bashrc</code></pre>

或者退出后重新登录。

<h3>3. 下载Alphafold 2源代码</h3>

<pre><code>git clone https://github.com/kuixu/alphafold.git</code></pre>
由于在校内访问github受限，可以在公网登录Alphafold 2代码页面
[https://github.com/kuixu/alphafold](https://github.com/kuixu/alphafold "https://github.com/kuixu/alphafold")
打包下载源代码，上传至服务器后解压缩

<pre><code>gunzip alphafold-main.zip</code></pre>

上述两种方法选其一即可。

<h3>4. 安装Alphafold 2</h3>

<pre><code>cd alphafold</code></pre>

或者

<pre><code>cd alphafold-main</code></pre>

经git安装用前者，代码包解压缩则用后者。

<h4>4.1 新建conda环境并激活</h4>

<pre><code>conda create -n af2 python=3.8 -y
conda activate af2
</code></pre>

<h4>4.2 从conda-forge中安装openmm</h4>

<pre><code>conda install -y -c conda-forge openmm=7.5.1 pdbfixer pip</code></pre>

<h4>4.3 从bioconda中安装生信专用的库</h4>

<pre><code>conda install -y -c bioconda hmmer hhsuite==3.3.0 kalign2</code></pre>

<h4>4.4 从anaconda中安装cudnn</h4>

<pre><code>conda install -y -c nvidia cudnn==8.0.4<code></pre>

<h4>4.5 安装requirements.txt的需求，必须一条命令执行，不能分开执行</h4>

<pre><code>pip3 install --upgrade pip \
  && pip3 install -r ./requirements.txt \
  && pip3 install --upgrade "jax[cuda111]" -f https://storage.googleapis.com/jax-releases/jax_releases.html \
  && pip3 install jaxlib==0.1.70+cuda111 -f https://storage.googleapis.com/jax-releases/jax_releases.html
<code></pre>

<h4>4.6 替换jax版本</h4>

<pre><code>pip3 install jax==0.2.18<code></pre>

<h4>4.7 给openmm打上补丁</h4>

<pre><code>work_path=$PWD
a=$(which python)
cd $(dirname $(dirname $a))/lib/python3.8/site-packages
patch -p0 < $work_path/docker/openmm.patch
<code></pre>

<h4>4.8 下载测试用的序列文件（可略过）</h4>

<pre><code>curl "https://predictioncenter.org/casp14/target.cgi?target=T1050&view=sequence" -o T1050.fasta</code></pre>


<h3>5. 设置数据文件和参数文件位置</h3>
由于数据文件的访问速度决定了Alphafold 2的运行速度，因此我们把数据文件放在了gpu13上的一块SSD硬盘上。
首先要登录到gpu13节点

<pre><code>bsub -q special -m gpu13 -R "rusage[ngpus_physical=1]" -Is /bin/bash</code></pre>
如果当前gpu13节点可用，这个命令将使我们登录到gpu13节点上，并且提供一个交互的环境。
建立指向数据文件目录的软连接并下载最新的参数文件

<pre><code>mkdir database
cd database
ln -s /mnt/u_20211111/data/bfd bfd
ln -s /mnt/u_20211111/data/mgnify mgnify
ln -s /mnt/u_20211111/data/pdb70 pdb70
ln -s /mnt/u_20211111/data/pdb_mmcif pdb_mmcif
ln -s /mnt/u_20211111/data/uniclust30 uniclust30
ln -s /mnt/u_20211111/data/uniref90 uniref90
mkdir params
cd params
wget https://storage.googleapis.com/alphafold/alphafold_params_2021-07-14.tar
tar xvf alphafold_params_2021-07-14.tar
cd ../..
</code></pre>

<h3>6. 运行Alphafold 2</h3>
修改run_alphafold.py文件中的DOWNLOAD_DIR和output_dir两个变量的值。
将DOWNLOAD_DIR指向database所在的目录，output_dir指向自定义的alphafold计算的输出目录。

运行测试作业

<pre><code>conda activate af2
python3 run_alphafold.py --fasta_paths=T1050.fasta
</code></pre>
