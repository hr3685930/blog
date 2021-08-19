# CDH坑点记录

> scp -r /opt/cloudera/parcel-repo/KAFKA-3.1.0-1.3.1.0.p0.35-el7.parcel* master-1:/opt/cloudera/parcel-repo/ && scp -r /opt/cloudera/parcel-repo/manifest.json master-1:/opt/cloudera/parcel-repo/ && scp -r /opt/cloudera/csd/* master-1:/opt/cloudera/csd/  

> scp -r /opt/cloudera/parcel-repo/KAFKA-3.1.0-1.3.1.0.p0.35-el7.parcel* master-2:/opt/cloudera/parcel-repo/ && scp -r /opt/cloudera/parcel-repo/manifest.json master-2:/opt/cloudera/parcel-repo/ && scp -r /opt/cloudera/csd/* master-2:/opt/cloudera/csd/  

> scp -r /opt/cloudera/parcel-repo/KAFKA-3.1.0-1.3.1.0.p0.35-el7.parcel* slave-1:/opt/cloudera/parcel-repo/ && scp -r /opt/cloudera/parcel-repo/manifest.json slave-1:/opt/cloudera/parcel-repo/ && scp -r /opt/cloudera/csd/* slave-1:/opt/cloudera/csd/  

> scp -r /opt/cloudera/parcel-repo/KAFKA-3.1.0-1.3.1.0.p0.35-el7.parcel* slave-2:/opt/cloudera/parcel-repo/ && scp -r /opt/cloudera/parcel-repo/manifest.json slave-2:/opt/cloudera/parcel-repo/ && scp -r /opt/cloudera/csd/* slave-2:/opt/cloudera/csd/  

> scp -r /opt/cloudera/parcel-repo/KAFKA-3.1.0-1.3.1.0.p0.35-el7.parcel* slave-3:/opt/cloudera/parcel-repo/ && scp -r /opt/cloudera/parcel-repo/manifest.json slave-3:/opt/cloudera/parcel-repo/ && scp -r /opt/cloudera/csd/* slave-3:/opt/cloudera/csd/  



![](CDH%E5%9D%91%E7%82%B9%E8%AE%B0%E5%BD%95/Pasted%20Graphic.png)

> kafka-topics —list —zookeeper slave-2:2181,slave-3:2181  
> kafka-console-producer —broker-list > slave-1:9092,slave-2:9092,slave-3:9092 —topic apilog  


![](CDH%E5%9D%91%E7%82%B9%E8%AE%B0%E5%BD%95/page28image31438176.png)
 
> su hdfs  
> hdfs dfs -chown root /user  