# Custom Blueprint System2.0
Document:Download[CustomBpSystem2.0EN.docx](https://github.com/user-attachments/files/24016389/CustomBpSystem2.0EN.docx)[CustomBpSystem2.0CH.docx](https://github.com/user-attachments/files/24016392/CustomBpSystem2.0CH.docx)
I.Quick Start
1. Open the CustomBpSystem folder.
<img width="471" height="206" alt="image" src="https://github.com/user-attachments/assets/71e50de0-0a76-4037-918c-1a9acac03ff8" />


2.Find the DemoMap
<img width="488" height="175" alt="image" src="https://github.com/user-attachments/assets/7dbac147-fbbf-4678-b406-0f42c7a4b6f8" />
Detailed Path: CustomBpSystem/DEMO/DEMOCharacters/Maps/DemoMap.DemoMap'


3.Walk near the cylinder and press Eto open the programming interface.
<img width="315" height="209" alt="image" src="https://github.com/user-attachments/assets/a7b5d14c-857e-4e29-bd70-719849637f20" />


4.Panel Button Explanation:
<img width="553" height="286" alt="image" src="https://github.com/user-attachments/assets/116e69c7-1660-4a8a-8f79-a71f5573483e" />
1.Click to save after connecting your nodes.
2.Start executing the node logic.
3.Close the programming interface.
4.Button to add a new variable.
5.Displays details of the selected variable.
6.Right-click to open the available nodes context menu. Click and hold the right mouse button to drag the panel.


5.Basic panel settings can be configured in DA_CBPDefault.
<img width="324" height="139" alt="image" src="https://github.com/user-attachments/assets/e4770787-7559-4104-a40e-8b3b48e2eaef" />
<img width="554" height="580" alt="image" src="https://github.com/user-attachments/assets/ae49d98f-4235-4eb0-87a2-278d0fffe66a" />
1.Set the panel zoom speed and panning range.
2.Configure node pin types and quantities.
3.Set input pin types.
4.Set output pin types.
5.Define the available node types for the system.


6.These two components need to be added to the corresponding actors.
<img width="467" height="179" alt="image" src="https://github.com/user-attachments/assets/50aaf486-ec78-4c90-8408-391e3d97ee5b" />
1.Add to the Blueprint you want to interact with (e.g., as shown in BP_InteractionDemo).
<img width="553" height="172" alt="image" src="https://github.com/user-attachments/assets/da2bbc04-a8ca-4d59-8f5d-55385c65551c" />
2.Add to your character's Blueprint (e.g., as shown in BP_DemoPawn).
<img width="553" height="223" alt="image" src="https://github.com/user-attachments/assets/d19853b9-6b1a-457d-a653-4c51c22a0729" />


II.Customizing Our Nodes

1. You need to open /CustomBpSystem/UMG/Enum/E_NodeType.E_NodeTypeand add your desired node type.
<img width="327" height="240" alt="image" src="https://github.com/user-attachments/assets/7c2a4fa3-161c-4e95-89b0-5403b525ed22" />


2. If you want a new variable type, you also need to modify these two enumerations.
<img width="473" height="90" alt="image" src="https://github.com/user-attachments/assets/10833c05-dddc-4f38-bb8e-ecc2e7cc5101" />
Also, update the VarBreaklogic in WBP_NodePin, WBP_Var, and WBP_ViewPanel.
You also need to modify the macros in /CustomBpSystem/UMG/ML_CBP.ML_CBP. This is not very complex; their functions are as indicated by their names.
<img width="305" height="254" alt="image" src="https://github.com/user-attachments/assets/831faf33-ac85-4680-9502-cf105710177d" />


3.You need to modify AC_CBP.
<img width="284" height="122" alt="image" src="https://github.com/user-attachments/assets/65c2b86b-516d-4736-902f-c6c74afe6d4a" />


4.Define your created node type in DetermineNodeType.
<img width="554" height="239" alt="image" src="https://github.com/user-attachments/assets/b3724d6b-563f-48e5-b40b-eb31b5aef195" />


5.Set your execution logic in fRunLogic.
<img width="553" height="337" alt="image" src="https://github.com/user-attachments/assets/14bb994d-15ed-43b8-a08a-82f7e1820aac" />
<img width="553" height="239" alt="image" src="https://github.com/user-attachments/assets/c3785b14-1fdb-4776-b1ce-6b1d5ec2b86b" />
NodeIndexis the index of the currently executing node. PinIndexis its output pin number.
1.Set the output pin data after your node's execution (You need to execute step 1 beforestep 2. Refer to other node examples for specifics).
2.Execute the next node linked to the pin.
3.Execute the next node linked to the pin using multithreading. This will call fMultiRunLogic, so you also need to modify the logic within fMultiRunLogic.


6.These Delaynodes represent the execution time. You can change them to 0for instant execution, or set them to any desired value.
<img width="357" height="415" alt="image" src="https://github.com/user-attachments/assets/d245c58f-af28-4d36-acb1-8c9f13715aac" />


7.It should be working now. More tutorials will be released later.
