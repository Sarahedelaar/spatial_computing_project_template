# **Planning: product**

### ** Metro diagram**
> The metro diagram is based on our design goals and the routing of the different users, which can be found in the section “Planning: Process”. All the dots represent spaces which are connected by lines with different thicknesses and colours. The colours of the lines represent the user of the route. This is a schematic representation and will not be the shape of our building.
<iframe src="https://drive.google.com/file/d/1lBGIS_da51KINb115S3rJYmzXYhSN_7V/preview" width="900px" height="600px"></iframe>
Fig. 18 Metro diagram

### **Adjacency matrix**
> This matrix shows the relations between spaces, being either 0 (= no connection) or 1 (=connection), thus defining if there is a connection between spaces or not. This data is based on the metro diagram and will be used as a base for the weighted relation matrix.
<iframe src="https://docs.google.com/spreadsheets/d/1tJdaOPOMVSKOOw2fiJ0Tsq2A5TuIsmYL/preview" style="width:100%;height:500px;"></iframe>
Fig. 19 Adjacency matrix

### **Relations between spaces**
> In this matrix the relations between spaces are presented, ranging from 0-1. This defines how strong the connection between certain spaces is and illustrates how close they want to be together. 1 being the strongest connection and 0 means no connection, these scores are based on the metro diagram and the weighted routings from the previous section. If space 1 was one space away from space 2 then the relation would be 0.50, if space 2 had been two spaces away then the score would be 0.25. If the distance is further than two then there is no connection and 0.00 will be the connection between spaces. The amount of times a route is being used is also taken into account.
<iframe src="https://docs.google.com/spreadsheets/d/1sOsGPwzE13D8FmNZD64akcNyAFRVVJvhgMh-XqF3d34/preview" style="width:100%;height:500px;"></iframe>
Fig. 20 Relations between spaces