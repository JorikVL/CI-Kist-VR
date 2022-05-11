# Documentation VR component
 ![image](https://user-images.githubusercontent.com/75835510/164457681-8c637f3c-672b-4f46-9071-a6a16a1e125a.png)

This application is a different version of the application shown here: https://github.com/JorikVL/iot-KIST-DigitalTwin . The main difference is that this application is made for VR. It shown where the sensors (who send us data) are placed and displays the last data it can get above it.
## Unity

The engine used to make this application is unity, it’s a game engine that supports 2D and 3D games as well as mobile applications and platforms for VR/AR. 

## How does it work

The map itself is called ArcGis map API made by a company called ESRI. It handles the loading of the map and has some built in VR functionalities. More info can be found at: their website: https://www.arcgis.com/ and their documentation page: https://doc.arcgis.com/ .
The code behind the application

## APICall
The APICall script is the bridge between the backend server and the application, it has A URL to ask for new data. Every 5 minutes it tries to get new data from every sensor, the maximum Is 20 sensors right now. It sends a web request to the URL of your choice and returns the data and sends it to the JSONReader script.

    using System.Collections;
    using UnityEngine;
    using System;
    using UnityEngine.Networking;
    using UnityEngine.UI;


    namespace Assets.APIScripts
    {
        public class APICall : MonoBehaviour
        {
            private static string DEFAULT_URL = "http://192.168.100.134:1880/Data1";
            string targetUrl = DEFAULT_URL; 
            public static JSONReader jsonReader;

            public void Start()
            {
                InvokeRepeating("GetData", 0, 300);
            }

            private IEnumerator RequestRoutine(string url, Action<string> callback = null)
            {
                var request = UnityWebRequest.Get(url);
                yield return request.SendWebRequest();
                var data = request.downloadHandler.text;
                callback?.Invoke(data);
            }

            private void ResponseCallback(string data)
            {
                jsonReader = this.GetComponent<JSONReader>();
                jsonReader.AddSensor(data);
            }

            public void ApiCall()
            {
                this.StartCoroutine(this.RequestRoutine(targetUrl, this.ResponseCallback));
            }

            public void GetData()
            {
                Debug.Log("GetData");
                for (int i = 1; i <= 20; i++)
                {
                    targetUrl = "http://192.168.100.134:1880/Data" + i;
                    try
                    {
                        ApiCall();
                    }
                    catch (System.Exception)
                    {
                        Debug.Log("APICall failed");
                    }
                }
            }
        }
    }

## JSONReader
The JSONReader contains the Sensor class, which is the way we format the data we get from the database. We also We make a list of all the data and call the list: ‘sensors’ so we can get this data in another script. If there is an error with adding the sensor, such as no data to add, the script will catch the error and display an error message.

    using System.Collections;
    using System.Collections.Generic;
    using UnityEngine;
    using UnityEngine.UI;
    using System;

    public class JSONReader : MonoBehaviour
    {
        [Serializable]
        public class Sensor
        {
            public string _id;
            public string Name;
            public double Latitude;
            public double Longtitude;
            public int battery;
            public int CO2;
            public int humidity;
            public int pm10;
            public int pm25;
            public int pressure;
            public int salinity;
            public int temp;
            public int tvox;
            public string time;
        }
        public List<Sensor> sensors = new List<Sensor>();

        public void AddSensor(string json)
        {
            try
            {
                Sensor newSensor = JsonUtility.FromJson<Sensor>(json);
                if (newSensor != null)
                {
                    sensors.Add(newSensor);
                    Debug.Log("Sensor added to list");
                }
                else
                {
                    Debug.Log("Sensor returned null");
                }
            }
            catch
            {
                Debug.Log("Add sensor to list failed!");
            }
        }
    } 
  
## ShowDataSensor
The ShowDataSensor is the script that displays the data we got from the database in unity. We start by invoking the manager and a JSONReader. We also invoking the invokeRepeating which means we call the script every 300 seconds to see if we have new data and then show the newest data available. We also initiate our manager which is the unity component that handles all our datascript, such as ShowDataSensor, JSONReader and APICall to get the data we got from the database. The DisplayForSensor function is the same for every sensor we have. We look for the sensor we need, and then we load in the data given by the sensors list from JSONReader. 
  
    using System;
    using System.Collections;
    using System.Collections.Generic;
    using UnityEngine;
    using UnityEngine.UI;
    using UnityEngine.Networking;


    namespace Assets.APIScripts
    {
        public class ShowDataSensor : MonoBehaviour
        {
            private GameObject manager;
            private JSONReader jsonReader;
            private CreateMappins createmappins;
            private APICall DataCall;
            public List<GameObject> Mysensors = new List<GameObject>();
            public GameObject theObject;

            private int Sensorindex = 0;

            public void Start()
            {
                InvokeRepeating("DisplayData", 1, 300);
                manager = GameObject.Find("Manager");
                jsonReader = manager.GetComponent<JSONReader>();
                createmappins = manager.GetComponent<CreateMappins>();
                Mysensors.Clear();
                Mysensors = createmappins.Mypins;
            }

            public void DisplayData()
            {
                Mysensors = createmappins.Mypins;
                if (APICall.jsonReader.sensors != null)
                {
                    foreach (JSONReader.Sensor sensor in APICall.jsonReader.sensors)
                    {
                        if (Sensorindex <= APICall.jsonReader.sensors.Count && sensor != null)
                        {
                                theObject = GameObject.FindWithTag(sensor.Name);
                            if (theObject != null)
                            {
                                theObject.GetComponentsInChildren<Canvas>()[0].GetComponentsInChildren<Text>()[0].text = "ID: " + sensor._id + "\nBattery: " + sensor.battery + "\tCO2: " + sensor.CO2 + "\nHumidity: " + sensor.humidity + "\tPm10: " + sensor.pm10 + "\nPm2.5: " + sensor.pm25 + "\tPressure: " + sensor.pressure + "\nSalinity: " + sensor.salinity + "\ttemp: " + sensor.temp + "\nTvox: " + sensor.tvox + "\tDate: " + sensor.time;

                            }
                        }
                    }
                    Debug.Log("created: " + Sensorindex);
                    Sensorindex++;
                    Debug.Log("index: " + Sensorindex);
                }
            }
        }
    }
## CameraFollow
The CameraFollowscript is used to have the data displayed by our sensors to be facing the player all the time, so its easily readable. In the script we first give the camera is has to follow, which is mostly the player. After that it’s as easy as making an update function where we continually give the target a new position, and then rotating it towards the camera. There are 2 versions of rotation, the first version has very sharp movement, the second ones have a smoother rotation.

    using System.Collections;
    using System.Collections.Generic;
    using UnityEngine;

    public class CameraFollow : MonoBehaviour
    {
        public Camera Camera2Follow;
        public float CameraDistance = 30F;
        public float smoothTime = 0.3F;
        private Vector3 velocity = Vector3.zero;
        private Transform target;

        void Awake()
        {
            target = Camera2Follow.transform;
        }

        void Update()
        {
            // Define my target position in front of the camera ->
            Vector3 targetPosition = target.TransformPoint(new Vector3(0, 0, CameraDistance));
            // version 1: my object's rotation is always facing to camera with no dampening  ->
            //transform.LookAt(transform.position + Camera2Follow.transform.rotation * Vector3.forward, Camera2Follow.transform.rotation * Vector3.up);

            // version 2 : my object's rotation isn't finished synchronously with the position smooth.damp ->
            transform.rotation = Quaternion.RotateTowards(transform.rotation, target.rotation, 35 * Time.deltaTime);
        }
    }

## CreateMappins
The createmappins script dynamically creates Mappins, which are the sensors on the map. Every 5 minutes the script will look if there are new sensors, it will destroy all of them and then place them back with all possible new sensors. The placement of the sensors is done by the arc GIS location component, which is given the coordinates provided by the sensors itself.  The script also makes your new map pin a child of the map component, otherwise it wont work.  

    using System.Collections;
    using System.Collections.Generic;
    using UnityEngine;
    using Esri.ArcGISMapsSDK.Components;
    using Esri.ArcGISMapsSDK.Utils.GeoCoord;
    using Esri.ArcGISMapsSDK.Utils.Math;

    public class CreateMappins : MonoBehaviour
    {
        public GameObject Pinprefab;
        public GameObject _Reader;
        public List<GameObject> Mypins = new List<GameObject>();
        public GameObject tobeParent;

        private GameObject manager;
        private JSONReader jsonReader;

        void Start()
        {
            manager = GameObject.Find("Manager");
            jsonReader = manager.GetComponent<JSONReader>();
            tobeParent = GameObject.Find("ArcGIS Map");
            if (tobeParent != null)
            {
                InvokeRepeating("Placemappins", 1, 100);
            }
        }
        public void Placemappins()
        {
            foreach (GameObject mappin in Mypins)
            {
                Destroy(mappin);
            }
            Mypins.Clear();
            foreach (JSONReader.Sensor sensor in jsonReader.sensors)
            {
                Debug.Log(sensor._id);
                if (sensor._id != null && sensor.Latitude != null && sensor.Longtitude != null)
                {
                    var mappin = Instantiate(Pinprefab, tobeParent.transform);
                    mappin.gameObject.tag = sensor.Name;
                    //set location of the sensors.
                    var LocationComp = mappin.GetComponent<ArcGISLocationComponent>();
                    LocationComp.Position = new LatLon(sensor.Latitude, sensor.Longtitude, 2500);
                    Mypins.Add(mappin);
                }
            }
        }
    }
  
## References:
Setting up a node-red server: https://github.com/AP-IT-GH/ci-infrastructure

Arc-GIS website: https://www.arcgis.com/

2D-Component: https://github.com/JorikVL/iot-KIST-DigitalTwin
