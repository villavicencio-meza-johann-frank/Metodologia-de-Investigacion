```python
 class Node:
    def __init__(self, index, demand, serv_time, time_window, lat, long):
        self.index = index
        self.demand = demand
        self.serv_time = serv_time
        self.time_window = time_window
        self.lat = lat
        self.long = long

    def __index__(self):
        return self.index

    def __hash__(self):
        return self.index

    def __repr__(self):
        return f"Customer <{self.index}>"
```


```python
from docplex.mp.model import Model

def build_VRPTW_problem(capacity, nodes, times, vehicles,**kwargs):
    
    
    #Parámetros
    customers = [cstmrs for cstmrs in nodes if cstmrs.index != 0] 
    #Big M
    bigM = 1000

    #Declaración del modelo
    m = Model(name = 'VRPTW', **kwargs)
    
    # ----- Variables -----
    #Arco cubierto por vehículo
    m.x_var = m.binary_var_cube(vehicles,nodes,nodes, name= 'route')
    #Start time for the service of a customer
    m.s_var = m.continuous_var_matrix(vehicles, nodes, name = 'service_time')

    # ----- Restricciones -----
    #La satisfacción del cliente
    m.covering_cts = []
    for i in customers: 
        cust_satisf_ct = m.sum(
            m.x_var[u,i,j] for j in nodes for u in vehicles)>= 1
        cust_satisf_ct.name = 'customer_satisf_{0!s}'.format(i.index)
        m.covering_cts.append(cust_satisf_ct)

    m.add_constraints(m.covering_cts)
     
    # Estructura de ruta
    m.flow_cts = []
    for i in nodes:
        for u in vehicles:
            flow_ct = (m.sum(m.x_var[u,i,j] for j in nodes) - 
                m.sum(m.x_var[u,j,i] for j in nodes)) == 0
            flow_ct.name = 'flow_constraint_{0!s}_{1!s}'.format(i.index,u)
            m.flow_cts.append(flow_ct)

    for u in vehicles: 
        flow_ct = m.sum(m.x_var[u,nodes[0],i] for i in nodes) <= 1
        flow_ct.name = 'flow_constraint_{0!s}_{1!s}_origin'.format(nodes[0].index,u)
        m.flow_cts.append(flow_ct)

    m.add_constraints(m.flow_cts)
 
    #Capacidad del vehículo
    m.capacity_cts = []
    for u in vehicles:
        capacity_ct = m.sum(nodes[i].demand * m.x_var[u,i,j] for i in nodes for j in nodes) <= capacity
        capacity_ct.name = 'capacity_vehicle{0!s}'.format(u)
        m.capacity_cts.append(capacity_ct)
    
    m.add_constraints(m.capacity_cts)
    
    #Ventanas de tiempo
    m.time_window_cts = []
    for i in nodes:
        for j in customers: 
            for u in vehicles:
             time_window_ct = m.s_var[u,i] + i.serv_time + times[i.index][j.index] - m.s_var[u,j] + bigM*m.x_var[u,i,j] <= bigM
             time_window_ct.name = 'time_window_{0!s}_{1!s}_{2!s}'.format(u,i.index,j.index)
             m.time_window_cts.append(time_window_ct)
    
    for i in customers:
        for u in vehicles: 
         time_window_ct = m.s_var[u,i]+ i.serv_time + times[i][0] - nodes[0].time_window[1] + bigM*m.x_var[u,i,nodes[0]]<= bigM
         time_window_ct.name = 'time_window_{0!s}_{1!s}_{2!s}_arrival'.format(u,i.index,nodes[0].index)
         m.time_window_cts.append(time_window_ct)

    for i in nodes:
        for u in vehicles:
            time_window_ct_lb = i.time_window[0] <= m.s_var[u,i]
            time_window_ct_lb.name = 'time_window_lb_{0!s}'.format(i.index)
            time_window_ct_ub = i.time_window[1] >= m.s_var[u,i]
            time_window_ct_ub.name = 'time_window_ub_{0!s}'.format(i.index)
            m.time_window_cts.append(time_window_ct_lb)
            m.time_window_cts.append(time_window_ct_ub)
    m.add_constraints(m.time_window_cts)
 
    #----- Función objetiva -----
    #Minimice el costo total
    m.total_cost = m.sum(times[i][j]*m.x_var[u,i,j] for i in nodes for j in nodes for u in vehicles)
    m.minimize(m.total_cost)
    
    return m

```


```python
CENTRO_REPARTO = Node(0,0,0,[0,120],-12.055286215862173, -77.03761770324066)# Wilson centro cómputo compras
Punto_1 = Node(1,10,3,[0,20],-11.860864774319099, -77.08270885357037)# ,Domicilio 1 Puente Piedra
Punto_2 = Node(2,20,3,[20,40], -11.880264510131493, -77.02095950290273)# ,Domicilio 2 Carabayllo 
Punto_3 = Node(3,30,3,[40,60],-11.98956118254249, -77.09880073186692)# Domicilio 3 San Martin de Porres
Punto_4 = Node(4,20,3,[0,80], -12.047362913132519, -77.12649612188682)# ,Domicilio 4  Callao
Punto_5 = Node(5,20,3,[60,80],-12.165407596126173, -76.96328741326177)#Domicilio 5 Miraflores
Punto_6 = Node(6,65,3,[80,120],-12.022785437170445, -76.89412395721087)#Domicilio 6 Ate
Punto_7 = Node(7,15,3,[80,120],-12.114866761848441, -77.03982184054135)#Domicilio 7 Miraflores
Punto_8 = Node(8,40,3,[80,120],-12.243772882472243, -76.91589482960883)#Domicilio 8 Villa el Salvador
vehicles = [1,2,3]
capacity = 100
nodes = [CENTRO_REPARTO, Punto_1,Punto_2, Punto_3,Punto_4,Punto_5,Punto_6,Punto_7,Punto_8]
```


```python
import folium
from folium import Marker

m = folium.Map(location=[-12.047209, -77.069313], tiles='openstreetmap', zoom_start=11)
for node in [n for n in nodes if n.index != 0]:
 Marker(location= [node.lat, node.long], icon= (folium.Icon(color='red', icon='home', prefix='fa'))).add_child(
    folium.Popup('Punto {}'.format(node.index))).add_to(m)

 Marker(location= [nodes[0].lat,nodes[0].long], icon= (folium.Icon(color='green', icon='truck', prefix='fa'))).add_child(
    folium.Popup('ZONA DE REPARTO')).add_to(m)
m
```




<div style="width:100%;"><div style="position:relative;width:100%;height:0;padding-bottom:60%;"><span style="color:#565656">Make this Notebook Trusted to load map: File -> Trust Notebook</span><iframe src="about:blank" style="position:absolute;width:100%;height:100%;left:0;top:0;border:none !important;" data-html=%3C%21DOCTYPE%20html%3E%0A%3Chead%3E%20%20%20%20%0A%20%20%20%20%3Cmeta%20http-equiv%3D%22content-type%22%20content%3D%22text/html%3B%20charset%3DUTF-8%22%20/%3E%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%3Cscript%3E%0A%20%20%20%20%20%20%20%20%20%20%20%20L_NO_TOUCH%20%3D%20false%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20L_DISABLE_3D%20%3D%20false%3B%0A%20%20%20%20%20%20%20%20%3C/script%3E%0A%20%20%20%20%0A%20%20%20%20%3Cstyle%3Ehtml%2C%20body%20%7Bwidth%3A%20100%25%3Bheight%3A%20100%25%3Bmargin%3A%200%3Bpadding%3A%200%3B%7D%3C/style%3E%0A%20%20%20%20%3Cstyle%3E%23map%20%7Bposition%3Aabsolute%3Btop%3A0%3Bbottom%3A0%3Bright%3A0%3Bleft%3A0%3B%7D%3C/style%3E%0A%20%20%20%20%3Cscript%20src%3D%22https%3A//cdn.jsdelivr.net/npm/leaflet%401.6.0/dist/leaflet.js%22%3E%3C/script%3E%0A%20%20%20%20%3Cscript%20src%3D%22https%3A//code.jquery.com/jquery-1.12.4.min.js%22%3E%3C/script%3E%0A%20%20%20%20%3Cscript%20src%3D%22https%3A//maxcdn.bootstrapcdn.com/bootstrap/3.2.0/js/bootstrap.min.js%22%3E%3C/script%3E%0A%20%20%20%20%3Cscript%20src%3D%22https%3A//cdnjs.cloudflare.com/ajax/libs/Leaflet.awesome-markers/2.0.2/leaflet.awesome-markers.js%22%3E%3C/script%3E%0A%20%20%20%20%3Clink%20rel%3D%22stylesheet%22%20href%3D%22https%3A//cdn.jsdelivr.net/npm/leaflet%401.6.0/dist/leaflet.css%22/%3E%0A%20%20%20%20%3Clink%20rel%3D%22stylesheet%22%20href%3D%22https%3A//maxcdn.bootstrapcdn.com/bootstrap/3.2.0/css/bootstrap.min.css%22/%3E%0A%20%20%20%20%3Clink%20rel%3D%22stylesheet%22%20href%3D%22https%3A//maxcdn.bootstrapcdn.com/bootstrap/3.2.0/css/bootstrap-theme.min.css%22/%3E%0A%20%20%20%20%3Clink%20rel%3D%22stylesheet%22%20href%3D%22https%3A//maxcdn.bootstrapcdn.com/font-awesome/4.6.3/css/font-awesome.min.css%22/%3E%0A%20%20%20%20%3Clink%20rel%3D%22stylesheet%22%20href%3D%22https%3A//cdnjs.cloudflare.com/ajax/libs/Leaflet.awesome-markers/2.0.2/leaflet.awesome-markers.css%22/%3E%0A%20%20%20%20%3Clink%20rel%3D%22stylesheet%22%20href%3D%22https%3A//cdn.jsdelivr.net/gh/python-visualization/folium/folium/templates/leaflet.awesome.rotate.min.css%22/%3E%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20%3Cmeta%20name%3D%22viewport%22%20content%3D%22width%3Ddevice-width%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20initial-scale%3D1.0%2C%20maximum-scale%3D1.0%2C%20user-scalable%3Dno%22%20/%3E%0A%20%20%20%20%20%20%20%20%20%20%20%20%3Cstyle%3E%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%23map_7e7235b90733497bb4fc2b184b91e26a%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20position%3A%20relative%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20width%3A%20100.0%25%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20height%3A%20100.0%25%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20left%3A%200.0%25%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20top%3A%200.0%25%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%3C/style%3E%0A%20%20%20%20%20%20%20%20%0A%3C/head%3E%0A%3Cbody%3E%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20%3Cdiv%20class%3D%22folium-map%22%20id%3D%22map_7e7235b90733497bb4fc2b184b91e26a%22%20%3E%3C/div%3E%0A%20%20%20%20%20%20%20%20%0A%3C/body%3E%0A%3Cscript%3E%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20map_7e7235b90733497bb4fc2b184b91e26a%20%3D%20L.map%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%22map_7e7235b90733497bb4fc2b184b91e26a%22%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20center%3A%20%5B-12.047209%2C%20-77.069313%5D%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20crs%3A%20L.CRS.EPSG3857%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20zoom%3A%2011%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20zoomControl%3A%20true%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20preferCanvas%3A%20false%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29%3B%0A%0A%20%20%20%20%20%20%20%20%20%20%20%20%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20tile_layer_5c7a14ea5c744e3c9318a93103de7645%20%3D%20L.tileLayer%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%22https%3A//%7Bs%7D.tile.openstreetmap.org/%7Bz%7D/%7Bx%7D/%7By%7D.png%22%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%22attribution%22%3A%20%22Data%20by%20%5Cu0026copy%3B%20%5Cu003ca%20href%3D%5C%22http%3A//openstreetmap.org%5C%22%5Cu003eOpenStreetMap%5Cu003c/a%5Cu003e%2C%20under%20%5Cu003ca%20href%3D%5C%22http%3A//www.openstreetmap.org/copyright%5C%22%5Cu003eODbL%5Cu003c/a%5Cu003e.%22%2C%20%22detectRetina%22%3A%20false%2C%20%22maxNativeZoom%22%3A%2018%2C%20%22maxZoom%22%3A%2018%2C%20%22minZoom%22%3A%200%2C%20%22noWrap%22%3A%20false%2C%20%22opacity%22%3A%201%2C%20%22subdomains%22%3A%20%22abc%22%2C%20%22tms%22%3A%20false%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29.addTo%28map_7e7235b90733497bb4fc2b184b91e26a%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20marker_906804034ca24d8397000012e4fc0aba%20%3D%20L.marker%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%5B-11.860864774319099%2C%20-77.08270885357037%5D%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29.addTo%28map_7e7235b90733497bb4fc2b184b91e26a%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20icon_582ba8902a914d0c9920cb8a46e39c88%20%3D%20L.AwesomeMarkers.icon%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%22extraClasses%22%3A%20%22fa-rotate-0%22%2C%20%22icon%22%3A%20%22home%22%2C%20%22iconColor%22%3A%20%22white%22%2C%20%22markerColor%22%3A%20%22red%22%2C%20%22prefix%22%3A%20%22fa%22%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20marker_906804034ca24d8397000012e4fc0aba.setIcon%28icon_582ba8902a914d0c9920cb8a46e39c88%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20var%20popup_c348f892214746d69af4aff4a59755b1%20%3D%20L.popup%28%7B%22maxWidth%22%3A%20%22100%25%22%7D%29%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20html_e245258bc6494be4ad1d1e56aace971c%20%3D%20%24%28%60%3Cdiv%20id%3D%22html_e245258bc6494be4ad1d1e56aace971c%22%20style%3D%22width%3A%20100.0%25%3B%20height%3A%20100.0%25%3B%22%3EPunto%201%3C/div%3E%60%29%5B0%5D%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20popup_c348f892214746d69af4aff4a59755b1.setContent%28html_e245258bc6494be4ad1d1e56aace971c%29%3B%0A%20%20%20%20%20%20%20%20%0A%0A%20%20%20%20%20%20%20%20marker_906804034ca24d8397000012e4fc0aba.bindPopup%28popup_c348f892214746d69af4aff4a59755b1%29%0A%20%20%20%20%20%20%20%20%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20marker_63b1d02139554c65b5768c2a3ddfe04f%20%3D%20L.marker%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%5B-12.055286215862173%2C%20-77.03761770324066%5D%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29.addTo%28map_7e7235b90733497bb4fc2b184b91e26a%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20icon_5cc2509041904639a80a0e40b9f2aca4%20%3D%20L.AwesomeMarkers.icon%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%22extraClasses%22%3A%20%22fa-rotate-0%22%2C%20%22icon%22%3A%20%22truck%22%2C%20%22iconColor%22%3A%20%22white%22%2C%20%22markerColor%22%3A%20%22green%22%2C%20%22prefix%22%3A%20%22fa%22%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20marker_63b1d02139554c65b5768c2a3ddfe04f.setIcon%28icon_5cc2509041904639a80a0e40b9f2aca4%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20var%20popup_ed6d861d6fa04dbeae57d7a8bb3bb3d0%20%3D%20L.popup%28%7B%22maxWidth%22%3A%20%22100%25%22%7D%29%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20html_060158ec080e46ffb18e9066f07e701a%20%3D%20%24%28%60%3Cdiv%20id%3D%22html_060158ec080e46ffb18e9066f07e701a%22%20style%3D%22width%3A%20100.0%25%3B%20height%3A%20100.0%25%3B%22%3EZONA%20DE%20REPARTO%3C/div%3E%60%29%5B0%5D%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20popup_ed6d861d6fa04dbeae57d7a8bb3bb3d0.setContent%28html_060158ec080e46ffb18e9066f07e701a%29%3B%0A%20%20%20%20%20%20%20%20%0A%0A%20%20%20%20%20%20%20%20marker_63b1d02139554c65b5768c2a3ddfe04f.bindPopup%28popup_ed6d861d6fa04dbeae57d7a8bb3bb3d0%29%0A%20%20%20%20%20%20%20%20%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20marker_684a9bffbe004122b09fba2e5d3d75a6%20%3D%20L.marker%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%5B-11.880264510131493%2C%20-77.02095950290273%5D%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29.addTo%28map_7e7235b90733497bb4fc2b184b91e26a%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20icon_43b8e93826d94dfca5037a03a8863ce5%20%3D%20L.AwesomeMarkers.icon%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%22extraClasses%22%3A%20%22fa-rotate-0%22%2C%20%22icon%22%3A%20%22home%22%2C%20%22iconColor%22%3A%20%22white%22%2C%20%22markerColor%22%3A%20%22red%22%2C%20%22prefix%22%3A%20%22fa%22%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20marker_684a9bffbe004122b09fba2e5d3d75a6.setIcon%28icon_43b8e93826d94dfca5037a03a8863ce5%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20var%20popup_2d27cb7f741a4b24b72291f4709fca6f%20%3D%20L.popup%28%7B%22maxWidth%22%3A%20%22100%25%22%7D%29%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20html_bb300bdd5028456b80834b465a4f0d6b%20%3D%20%24%28%60%3Cdiv%20id%3D%22html_bb300bdd5028456b80834b465a4f0d6b%22%20style%3D%22width%3A%20100.0%25%3B%20height%3A%20100.0%25%3B%22%3EPunto%202%3C/div%3E%60%29%5B0%5D%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20popup_2d27cb7f741a4b24b72291f4709fca6f.setContent%28html_bb300bdd5028456b80834b465a4f0d6b%29%3B%0A%20%20%20%20%20%20%20%20%0A%0A%20%20%20%20%20%20%20%20marker_684a9bffbe004122b09fba2e5d3d75a6.bindPopup%28popup_2d27cb7f741a4b24b72291f4709fca6f%29%0A%20%20%20%20%20%20%20%20%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20marker_a7c3569bd43341599c639bf73c3b7eca%20%3D%20L.marker%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%5B-12.055286215862173%2C%20-77.03761770324066%5D%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29.addTo%28map_7e7235b90733497bb4fc2b184b91e26a%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20icon_1918fb4986544ad7aa398742946e1b6e%20%3D%20L.AwesomeMarkers.icon%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%22extraClasses%22%3A%20%22fa-rotate-0%22%2C%20%22icon%22%3A%20%22truck%22%2C%20%22iconColor%22%3A%20%22white%22%2C%20%22markerColor%22%3A%20%22green%22%2C%20%22prefix%22%3A%20%22fa%22%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20marker_a7c3569bd43341599c639bf73c3b7eca.setIcon%28icon_1918fb4986544ad7aa398742946e1b6e%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20var%20popup_dd9cb1873819498792b35604a9ebbc28%20%3D%20L.popup%28%7B%22maxWidth%22%3A%20%22100%25%22%7D%29%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20html_e19b4fb9371b476eb4100062f698f102%20%3D%20%24%28%60%3Cdiv%20id%3D%22html_e19b4fb9371b476eb4100062f698f102%22%20style%3D%22width%3A%20100.0%25%3B%20height%3A%20100.0%25%3B%22%3EZONA%20DE%20REPARTO%3C/div%3E%60%29%5B0%5D%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20popup_dd9cb1873819498792b35604a9ebbc28.setContent%28html_e19b4fb9371b476eb4100062f698f102%29%3B%0A%20%20%20%20%20%20%20%20%0A%0A%20%20%20%20%20%20%20%20marker_a7c3569bd43341599c639bf73c3b7eca.bindPopup%28popup_dd9cb1873819498792b35604a9ebbc28%29%0A%20%20%20%20%20%20%20%20%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20marker_647f3f38b3d549edb7c66c58663a7441%20%3D%20L.marker%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%5B-11.98956118254249%2C%20-77.09880073186692%5D%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29.addTo%28map_7e7235b90733497bb4fc2b184b91e26a%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20icon_9d0a5a77f7804433a1f246f11e0b6d60%20%3D%20L.AwesomeMarkers.icon%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%22extraClasses%22%3A%20%22fa-rotate-0%22%2C%20%22icon%22%3A%20%22home%22%2C%20%22iconColor%22%3A%20%22white%22%2C%20%22markerColor%22%3A%20%22red%22%2C%20%22prefix%22%3A%20%22fa%22%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20marker_647f3f38b3d549edb7c66c58663a7441.setIcon%28icon_9d0a5a77f7804433a1f246f11e0b6d60%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20var%20popup_a3dbe3ca566940eb8c34cc03fd872867%20%3D%20L.popup%28%7B%22maxWidth%22%3A%20%22100%25%22%7D%29%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20html_248c08cc810748e89ff7b488991b8d4d%20%3D%20%24%28%60%3Cdiv%20id%3D%22html_248c08cc810748e89ff7b488991b8d4d%22%20style%3D%22width%3A%20100.0%25%3B%20height%3A%20100.0%25%3B%22%3EPunto%203%3C/div%3E%60%29%5B0%5D%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20popup_a3dbe3ca566940eb8c34cc03fd872867.setContent%28html_248c08cc810748e89ff7b488991b8d4d%29%3B%0A%20%20%20%20%20%20%20%20%0A%0A%20%20%20%20%20%20%20%20marker_647f3f38b3d549edb7c66c58663a7441.bindPopup%28popup_a3dbe3ca566940eb8c34cc03fd872867%29%0A%20%20%20%20%20%20%20%20%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20marker_cf0c1abc900640e1b3dc091695a6ac0f%20%3D%20L.marker%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%5B-12.055286215862173%2C%20-77.03761770324066%5D%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29.addTo%28map_7e7235b90733497bb4fc2b184b91e26a%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20icon_d1b65c7cb9784c6282e889458d00a95b%20%3D%20L.AwesomeMarkers.icon%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%22extraClasses%22%3A%20%22fa-rotate-0%22%2C%20%22icon%22%3A%20%22truck%22%2C%20%22iconColor%22%3A%20%22white%22%2C%20%22markerColor%22%3A%20%22green%22%2C%20%22prefix%22%3A%20%22fa%22%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20marker_cf0c1abc900640e1b3dc091695a6ac0f.setIcon%28icon_d1b65c7cb9784c6282e889458d00a95b%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20var%20popup_6f48c94c6e1d42c3a5a45473416ab168%20%3D%20L.popup%28%7B%22maxWidth%22%3A%20%22100%25%22%7D%29%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20html_312e16cd77e2460793653c5f707daa50%20%3D%20%24%28%60%3Cdiv%20id%3D%22html_312e16cd77e2460793653c5f707daa50%22%20style%3D%22width%3A%20100.0%25%3B%20height%3A%20100.0%25%3B%22%3EZONA%20DE%20REPARTO%3C/div%3E%60%29%5B0%5D%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20popup_6f48c94c6e1d42c3a5a45473416ab168.setContent%28html_312e16cd77e2460793653c5f707daa50%29%3B%0A%20%20%20%20%20%20%20%20%0A%0A%20%20%20%20%20%20%20%20marker_cf0c1abc900640e1b3dc091695a6ac0f.bindPopup%28popup_6f48c94c6e1d42c3a5a45473416ab168%29%0A%20%20%20%20%20%20%20%20%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20marker_b2153f42183e4446993d5da86c7dcbdf%20%3D%20L.marker%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%5B-12.047362913132519%2C%20-77.12649612188682%5D%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29.addTo%28map_7e7235b90733497bb4fc2b184b91e26a%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20icon_d09d1127f9af4c8a825e5509e7d24bf7%20%3D%20L.AwesomeMarkers.icon%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%22extraClasses%22%3A%20%22fa-rotate-0%22%2C%20%22icon%22%3A%20%22home%22%2C%20%22iconColor%22%3A%20%22white%22%2C%20%22markerColor%22%3A%20%22red%22%2C%20%22prefix%22%3A%20%22fa%22%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20marker_b2153f42183e4446993d5da86c7dcbdf.setIcon%28icon_d09d1127f9af4c8a825e5509e7d24bf7%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20var%20popup_015bc977178043a8acea47e07444c05f%20%3D%20L.popup%28%7B%22maxWidth%22%3A%20%22100%25%22%7D%29%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20html_c08be15cb0954c91a733483f88cb297d%20%3D%20%24%28%60%3Cdiv%20id%3D%22html_c08be15cb0954c91a733483f88cb297d%22%20style%3D%22width%3A%20100.0%25%3B%20height%3A%20100.0%25%3B%22%3EPunto%204%3C/div%3E%60%29%5B0%5D%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20popup_015bc977178043a8acea47e07444c05f.setContent%28html_c08be15cb0954c91a733483f88cb297d%29%3B%0A%20%20%20%20%20%20%20%20%0A%0A%20%20%20%20%20%20%20%20marker_b2153f42183e4446993d5da86c7dcbdf.bindPopup%28popup_015bc977178043a8acea47e07444c05f%29%0A%20%20%20%20%20%20%20%20%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20marker_b0db414bcf53469daff2ffd38c03b8ed%20%3D%20L.marker%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%5B-12.055286215862173%2C%20-77.03761770324066%5D%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29.addTo%28map_7e7235b90733497bb4fc2b184b91e26a%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20icon_5493daf7339149dfb8c4c6c40967aa04%20%3D%20L.AwesomeMarkers.icon%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%22extraClasses%22%3A%20%22fa-rotate-0%22%2C%20%22icon%22%3A%20%22truck%22%2C%20%22iconColor%22%3A%20%22white%22%2C%20%22markerColor%22%3A%20%22green%22%2C%20%22prefix%22%3A%20%22fa%22%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20marker_b0db414bcf53469daff2ffd38c03b8ed.setIcon%28icon_5493daf7339149dfb8c4c6c40967aa04%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20var%20popup_4ab95fc31b474ece8089e1bf07df1968%20%3D%20L.popup%28%7B%22maxWidth%22%3A%20%22100%25%22%7D%29%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20html_b6d3a32e5a42462b98c017b3d5ed97b9%20%3D%20%24%28%60%3Cdiv%20id%3D%22html_b6d3a32e5a42462b98c017b3d5ed97b9%22%20style%3D%22width%3A%20100.0%25%3B%20height%3A%20100.0%25%3B%22%3EZONA%20DE%20REPARTO%3C/div%3E%60%29%5B0%5D%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20popup_4ab95fc31b474ece8089e1bf07df1968.setContent%28html_b6d3a32e5a42462b98c017b3d5ed97b9%29%3B%0A%20%20%20%20%20%20%20%20%0A%0A%20%20%20%20%20%20%20%20marker_b0db414bcf53469daff2ffd38c03b8ed.bindPopup%28popup_4ab95fc31b474ece8089e1bf07df1968%29%0A%20%20%20%20%20%20%20%20%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20marker_d9358de73ddc45d1abe53dcb4a503096%20%3D%20L.marker%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%5B-12.165407596126173%2C%20-76.96328741326177%5D%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29.addTo%28map_7e7235b90733497bb4fc2b184b91e26a%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20icon_00283b6ecbeb40bc8f3b84855f66ea32%20%3D%20L.AwesomeMarkers.icon%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%22extraClasses%22%3A%20%22fa-rotate-0%22%2C%20%22icon%22%3A%20%22home%22%2C%20%22iconColor%22%3A%20%22white%22%2C%20%22markerColor%22%3A%20%22red%22%2C%20%22prefix%22%3A%20%22fa%22%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20marker_d9358de73ddc45d1abe53dcb4a503096.setIcon%28icon_00283b6ecbeb40bc8f3b84855f66ea32%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20var%20popup_d0dbb3018bfd483384a9c86eda570259%20%3D%20L.popup%28%7B%22maxWidth%22%3A%20%22100%25%22%7D%29%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20html_e93dc3d9011d4c1faa8fe390170c45a0%20%3D%20%24%28%60%3Cdiv%20id%3D%22html_e93dc3d9011d4c1faa8fe390170c45a0%22%20style%3D%22width%3A%20100.0%25%3B%20height%3A%20100.0%25%3B%22%3EPunto%205%3C/div%3E%60%29%5B0%5D%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20popup_d0dbb3018bfd483384a9c86eda570259.setContent%28html_e93dc3d9011d4c1faa8fe390170c45a0%29%3B%0A%20%20%20%20%20%20%20%20%0A%0A%20%20%20%20%20%20%20%20marker_d9358de73ddc45d1abe53dcb4a503096.bindPopup%28popup_d0dbb3018bfd483384a9c86eda570259%29%0A%20%20%20%20%20%20%20%20%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20marker_27faa8c3ccf74aca9589b77b8ba31524%20%3D%20L.marker%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%5B-12.055286215862173%2C%20-77.03761770324066%5D%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29.addTo%28map_7e7235b90733497bb4fc2b184b91e26a%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20icon_bc33f1d76f374a7a863e96132d4b1f31%20%3D%20L.AwesomeMarkers.icon%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%22extraClasses%22%3A%20%22fa-rotate-0%22%2C%20%22icon%22%3A%20%22truck%22%2C%20%22iconColor%22%3A%20%22white%22%2C%20%22markerColor%22%3A%20%22green%22%2C%20%22prefix%22%3A%20%22fa%22%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20marker_27faa8c3ccf74aca9589b77b8ba31524.setIcon%28icon_bc33f1d76f374a7a863e96132d4b1f31%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20var%20popup_0f83f62c2a5b4ca6a9a30150ba846e3c%20%3D%20L.popup%28%7B%22maxWidth%22%3A%20%22100%25%22%7D%29%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20html_d0b1529ed1e140418e11ea45e5bef844%20%3D%20%24%28%60%3Cdiv%20id%3D%22html_d0b1529ed1e140418e11ea45e5bef844%22%20style%3D%22width%3A%20100.0%25%3B%20height%3A%20100.0%25%3B%22%3EZONA%20DE%20REPARTO%3C/div%3E%60%29%5B0%5D%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20popup_0f83f62c2a5b4ca6a9a30150ba846e3c.setContent%28html_d0b1529ed1e140418e11ea45e5bef844%29%3B%0A%20%20%20%20%20%20%20%20%0A%0A%20%20%20%20%20%20%20%20marker_27faa8c3ccf74aca9589b77b8ba31524.bindPopup%28popup_0f83f62c2a5b4ca6a9a30150ba846e3c%29%0A%20%20%20%20%20%20%20%20%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20marker_fde262eec9dc4f43abcb48b4da1ec27d%20%3D%20L.marker%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%5B-12.022785437170445%2C%20-76.89412395721087%5D%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29.addTo%28map_7e7235b90733497bb4fc2b184b91e26a%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20icon_ad43221cc8564027938b796de2363586%20%3D%20L.AwesomeMarkers.icon%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%22extraClasses%22%3A%20%22fa-rotate-0%22%2C%20%22icon%22%3A%20%22home%22%2C%20%22iconColor%22%3A%20%22white%22%2C%20%22markerColor%22%3A%20%22red%22%2C%20%22prefix%22%3A%20%22fa%22%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20marker_fde262eec9dc4f43abcb48b4da1ec27d.setIcon%28icon_ad43221cc8564027938b796de2363586%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20var%20popup_3b628a13824a446cb6eea6afbfd4fe28%20%3D%20L.popup%28%7B%22maxWidth%22%3A%20%22100%25%22%7D%29%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20html_1a63a57f1d774b7cbbec6bd09e9bd025%20%3D%20%24%28%60%3Cdiv%20id%3D%22html_1a63a57f1d774b7cbbec6bd09e9bd025%22%20style%3D%22width%3A%20100.0%25%3B%20height%3A%20100.0%25%3B%22%3EPunto%206%3C/div%3E%60%29%5B0%5D%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20popup_3b628a13824a446cb6eea6afbfd4fe28.setContent%28html_1a63a57f1d774b7cbbec6bd09e9bd025%29%3B%0A%20%20%20%20%20%20%20%20%0A%0A%20%20%20%20%20%20%20%20marker_fde262eec9dc4f43abcb48b4da1ec27d.bindPopup%28popup_3b628a13824a446cb6eea6afbfd4fe28%29%0A%20%20%20%20%20%20%20%20%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20marker_6125c2dfd3514d498d3d2d20aa3d3819%20%3D%20L.marker%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%5B-12.055286215862173%2C%20-77.03761770324066%5D%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29.addTo%28map_7e7235b90733497bb4fc2b184b91e26a%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20icon_838abaebca0a44eca393f95268355ed6%20%3D%20L.AwesomeMarkers.icon%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%22extraClasses%22%3A%20%22fa-rotate-0%22%2C%20%22icon%22%3A%20%22truck%22%2C%20%22iconColor%22%3A%20%22white%22%2C%20%22markerColor%22%3A%20%22green%22%2C%20%22prefix%22%3A%20%22fa%22%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20marker_6125c2dfd3514d498d3d2d20aa3d3819.setIcon%28icon_838abaebca0a44eca393f95268355ed6%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20var%20popup_6e2717a5db5f4bf79fac558950398f5e%20%3D%20L.popup%28%7B%22maxWidth%22%3A%20%22100%25%22%7D%29%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20html_1fa512ba96344a32974619bb554136b8%20%3D%20%24%28%60%3Cdiv%20id%3D%22html_1fa512ba96344a32974619bb554136b8%22%20style%3D%22width%3A%20100.0%25%3B%20height%3A%20100.0%25%3B%22%3EZONA%20DE%20REPARTO%3C/div%3E%60%29%5B0%5D%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20popup_6e2717a5db5f4bf79fac558950398f5e.setContent%28html_1fa512ba96344a32974619bb554136b8%29%3B%0A%20%20%20%20%20%20%20%20%0A%0A%20%20%20%20%20%20%20%20marker_6125c2dfd3514d498d3d2d20aa3d3819.bindPopup%28popup_6e2717a5db5f4bf79fac558950398f5e%29%0A%20%20%20%20%20%20%20%20%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20marker_e80a3ed7b0ce41b580f329f47baeec35%20%3D%20L.marker%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%5B-12.114866761848441%2C%20-77.03982184054135%5D%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29.addTo%28map_7e7235b90733497bb4fc2b184b91e26a%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20icon_3788edc498bf4efa8f2ec986dfff442b%20%3D%20L.AwesomeMarkers.icon%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%22extraClasses%22%3A%20%22fa-rotate-0%22%2C%20%22icon%22%3A%20%22home%22%2C%20%22iconColor%22%3A%20%22white%22%2C%20%22markerColor%22%3A%20%22red%22%2C%20%22prefix%22%3A%20%22fa%22%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20marker_e80a3ed7b0ce41b580f329f47baeec35.setIcon%28icon_3788edc498bf4efa8f2ec986dfff442b%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20var%20popup_34e58664594f4feb9fa1d1e48e222c41%20%3D%20L.popup%28%7B%22maxWidth%22%3A%20%22100%25%22%7D%29%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20html_7f8830f035124704ad5f21140bd4ebfe%20%3D%20%24%28%60%3Cdiv%20id%3D%22html_7f8830f035124704ad5f21140bd4ebfe%22%20style%3D%22width%3A%20100.0%25%3B%20height%3A%20100.0%25%3B%22%3EPunto%207%3C/div%3E%60%29%5B0%5D%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20popup_34e58664594f4feb9fa1d1e48e222c41.setContent%28html_7f8830f035124704ad5f21140bd4ebfe%29%3B%0A%20%20%20%20%20%20%20%20%0A%0A%20%20%20%20%20%20%20%20marker_e80a3ed7b0ce41b580f329f47baeec35.bindPopup%28popup_34e58664594f4feb9fa1d1e48e222c41%29%0A%20%20%20%20%20%20%20%20%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20marker_443f02c49e284ba3ad9cb842255094f6%20%3D%20L.marker%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%5B-12.055286215862173%2C%20-77.03761770324066%5D%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29.addTo%28map_7e7235b90733497bb4fc2b184b91e26a%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20icon_290220a9fcda4ae1aefbfedc26ce68e7%20%3D%20L.AwesomeMarkers.icon%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%22extraClasses%22%3A%20%22fa-rotate-0%22%2C%20%22icon%22%3A%20%22truck%22%2C%20%22iconColor%22%3A%20%22white%22%2C%20%22markerColor%22%3A%20%22green%22%2C%20%22prefix%22%3A%20%22fa%22%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20marker_443f02c49e284ba3ad9cb842255094f6.setIcon%28icon_290220a9fcda4ae1aefbfedc26ce68e7%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20var%20popup_554b3f8532494859a98e09e77e9fe99d%20%3D%20L.popup%28%7B%22maxWidth%22%3A%20%22100%25%22%7D%29%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20html_db1f74be8b58442c90a4301dd4a2c7be%20%3D%20%24%28%60%3Cdiv%20id%3D%22html_db1f74be8b58442c90a4301dd4a2c7be%22%20style%3D%22width%3A%20100.0%25%3B%20height%3A%20100.0%25%3B%22%3EZONA%20DE%20REPARTO%3C/div%3E%60%29%5B0%5D%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20popup_554b3f8532494859a98e09e77e9fe99d.setContent%28html_db1f74be8b58442c90a4301dd4a2c7be%29%3B%0A%20%20%20%20%20%20%20%20%0A%0A%20%20%20%20%20%20%20%20marker_443f02c49e284ba3ad9cb842255094f6.bindPopup%28popup_554b3f8532494859a98e09e77e9fe99d%29%0A%20%20%20%20%20%20%20%20%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20marker_e0d36403f4bf46e3913081419c6b27e9%20%3D%20L.marker%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%5B-12.243772882472243%2C%20-76.91589482960883%5D%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29.addTo%28map_7e7235b90733497bb4fc2b184b91e26a%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20icon_ff929cc7d82f46f4ab890b295fa91c75%20%3D%20L.AwesomeMarkers.icon%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%22extraClasses%22%3A%20%22fa-rotate-0%22%2C%20%22icon%22%3A%20%22home%22%2C%20%22iconColor%22%3A%20%22white%22%2C%20%22markerColor%22%3A%20%22red%22%2C%20%22prefix%22%3A%20%22fa%22%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20marker_e0d36403f4bf46e3913081419c6b27e9.setIcon%28icon_ff929cc7d82f46f4ab890b295fa91c75%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20var%20popup_f20d583b465a4b08b643c02d7ec9d36e%20%3D%20L.popup%28%7B%22maxWidth%22%3A%20%22100%25%22%7D%29%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20html_77b6a9ea222841d492d1eea2b9441751%20%3D%20%24%28%60%3Cdiv%20id%3D%22html_77b6a9ea222841d492d1eea2b9441751%22%20style%3D%22width%3A%20100.0%25%3B%20height%3A%20100.0%25%3B%22%3EPunto%208%3C/div%3E%60%29%5B0%5D%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20popup_f20d583b465a4b08b643c02d7ec9d36e.setContent%28html_77b6a9ea222841d492d1eea2b9441751%29%3B%0A%20%20%20%20%20%20%20%20%0A%0A%20%20%20%20%20%20%20%20marker_e0d36403f4bf46e3913081419c6b27e9.bindPopup%28popup_f20d583b465a4b08b643c02d7ec9d36e%29%0A%20%20%20%20%20%20%20%20%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20marker_64267ada19c04807b26a1d61c9f6a0ac%20%3D%20L.marker%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%5B-12.055286215862173%2C%20-77.03761770324066%5D%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29.addTo%28map_7e7235b90733497bb4fc2b184b91e26a%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20icon_c5d062f33299443789ac87df14959a42%20%3D%20L.AwesomeMarkers.icon%28%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7B%22extraClasses%22%3A%20%22fa-rotate-0%22%2C%20%22icon%22%3A%20%22truck%22%2C%20%22iconColor%22%3A%20%22white%22%2C%20%22markerColor%22%3A%20%22green%22%2C%20%22prefix%22%3A%20%22fa%22%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20marker_64267ada19c04807b26a1d61c9f6a0ac.setIcon%28icon_c5d062f33299443789ac87df14959a42%29%3B%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%20%20%20%20%20%20%20%20var%20popup_c474ec93c26a497c9201958cdde0b861%20%3D%20L.popup%28%7B%22maxWidth%22%3A%20%22100%25%22%7D%29%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%20%20%20%20%20%20%20%20var%20html_1565bc14467440c49477bcb5092071c6%20%3D%20%24%28%60%3Cdiv%20id%3D%22html_1565bc14467440c49477bcb5092071c6%22%20style%3D%22width%3A%20100.0%25%3B%20height%3A%20100.0%25%3B%22%3EZONA%20DE%20REPARTO%3C/div%3E%60%29%5B0%5D%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20popup_c474ec93c26a497c9201958cdde0b861.setContent%28html_1565bc14467440c49477bcb5092071c6%29%3B%0A%20%20%20%20%20%20%20%20%0A%0A%20%20%20%20%20%20%20%20marker_64267ada19c04807b26a1d61c9f6a0ac.bindPopup%28popup_c474ec93c26a497c9201958cdde0b861%29%0A%20%20%20%20%20%20%20%20%3B%0A%0A%20%20%20%20%20%20%20%20%0A%20%20%20%20%0A%3C/script%3E onload="this.contentDocument.open();this.contentDocument.write(    decodeURIComponent(this.getAttribute('data-html')));this.contentDocument.close();" allowfullscreen webkitallowfullscreen mozallowfullscreen></iframe></div></div>




```python

times = [[round(((node_origin.lat - node_destin.lat)**2 + 
    (node_origin.long - node_destin.long)**2)**(1/2),3)*100 
    for node_destin in nodes] for node_origin in nodes]
times
```




    [[0.0,
      20.0,
      17.599999999999998,
      9.0,
      8.9,
      13.3,
      14.7,
      6.0,
      22.400000000000002],
     [20.0, 0.0, 6.5, 13.0, 19.2, 32.7, 24.9, 25.8, 41.8],
     [17.599999999999998,
      6.5,
      0.0,
      13.4,
      19.8,
      29.099999999999998,
      19.1,
      23.5,
      37.8],
     [9.0, 13.0, 13.4, 0.0, 6.4, 22.2, 20.7, 13.8, 31.3],
     [8.9,
      19.2,
      19.8,
      6.4,
      0.0,
      20.1,
      23.400000000000002,
      11.0,
      28.799999999999997],
     [13.3, 32.7, 29.099999999999998, 22.2, 20.1, 0.0, 15.9, 9.2, 9.2],
     [14.7, 24.9, 19.1, 20.7, 23.400000000000002, 15.9, 0.0, 17.2, 22.2],
     [6.0, 25.8, 23.5, 13.8, 11.0, 9.2, 17.2, 0.0, 17.9],
     [22.400000000000002,
      41.8,
      37.8,
      31.3,
      28.799999999999997,
      9.2,
      22.2,
      17.9,
      0.0]]




```python
mod = build_VRPTW_problem(capacity, nodes, times, vehicles)
```


```python
import json 
sol = mod.solve()
print (sol)

```

    solution for: VRPTW
    objective: 131
    route_1_Customer <0>_Customer <5>=1
    route_1_Customer <5>_Customer <8>=1
    route_1_Customer <7>_Customer <0>=1
    route_1_Customer <8>_Customer <7>=1
    route_2_Customer <0>_Customer <1>=1
    route_2_Customer <1>_Customer <2>=1
    route_2_Customer <2>_Customer <6>=1
    route_2_Customer <6>_Customer <0>=1
    route_3_Customer <0>_Customer <3>=1
    route_3_Customer <3>_Customer <4>=1
    route_3_Customer <4>_Customer <0>=1
    service_time_1_Customer <1>=-0.000
    service_time_1_Customer <2>=20.000
    service_time_1_Customer <3>=40.000
    service_time_1_Customer <5>=61.100
    service_time_1_Customer <6>=80.000
    service_time_1_Customer <7>=100.900
    service_time_1_Customer <8>=80.000
    service_time_2_Customer <1>=20.000
    service_time_2_Customer <2>=29.500
    service_time_2_Customer <3>=40.000
    service_time_2_Customer <5>=61.100
    service_time_2_Customer <6>=80.000
    service_time_2_Customer <7>=80.000
    service_time_2_Customer <8>=80.000
    service_time_3_Customer <1>=-0.000
    service_time_3_Customer <2>=20.000
    service_time_3_Customer <3>=40.000
    service_time_3_Customer <4>=69.400
    service_time_3_Customer <5>=61.100
    service_time_3_Customer <6>=80.000
    service_time_3_Customer <7>=80.000
    service_time_3_Customer <8>=80.000
    
    


```python
sol_dict = json.loads(sol.export_as_json_string())

sol_dict['CPLEXSolution']['variables'][0:16]

```




    [{'index': '5', 'name': 'route_1_Customer <0>_Customer <5>', 'value': '1.0'},
     {'index': '53', 'name': 'route_1_Customer <5>_Customer <8>', 'value': '1.0'},
     {'index': '63', 'name': 'route_1_Customer <7>_Customer <0>', 'value': '1.0'},
     {'index': '79', 'name': 'route_1_Customer <8>_Customer <7>', 'value': '1.0'},
     {'index': '82', 'name': 'route_2_Customer <0>_Customer <1>', 'value': '1.0'},
     {'index': '92', 'name': 'route_2_Customer <1>_Customer <2>', 'value': '1.0'},
     {'index': '105', 'name': 'route_2_Customer <2>_Customer <6>', 'value': '1.0'},
     {'index': '135', 'name': 'route_2_Customer <6>_Customer <0>', 'value': '1.0'},
     {'index': '165', 'name': 'route_3_Customer <0>_Customer <3>', 'value': '1.0'},
     {'index': '193', 'name': 'route_3_Customer <3>_Customer <4>', 'value': '1.0'},
     {'index': '198', 'name': 'route_3_Customer <4>_Customer <0>', 'value': '1.0'},
     {'index': '244',
      'name': 'service_time_1_Customer <1>',
      'value': '-2.2737367544323206e-13'},
     {'index': '245', 'name': 'service_time_1_Customer <2>', 'value': '20.0'},
     {'index': '246', 'name': 'service_time_1_Customer <3>', 'value': '40.0'},
     {'index': '248',
      'name': 'service_time_1_Customer <5>',
      'value': '61.09999999999998'},
     {'index': '249', 'name': 'service_time_1_Customer <6>', 'value': '80.0'}]




```python

```


```python

```
