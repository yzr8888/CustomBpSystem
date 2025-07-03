# CustomBpSystem
Document:Download[Custom Blueprint SystemEN.docx](https://github.com/user-attachments/files/21033451/Custom.Blueprint.SystemEN.docx)
[Custom Blueprint SystemZH.docx…]()
1.How to enter the demo demonstration?
Open Custom Blueprint System/DEMO/Map/DemoMap
After entering theexample
Key 1:Create BPNode.
Right-click node:Open dropdown menu.
Hold Left-click + Drag:Pan view.
Mouse Wheel:Zoom view.
Left-click Output Pin:Start drawing connection.
Right-click Input Pin:Connect to pin (completes connection started with left-click on output pin).

2.How to add a custom BPNode?

(1)Open Custom Blueprint System/WBP/Enum/E_BPNodeType. Add a custom enumeration name for you.

![image](https://github.com/user-attachments/assets/1addce18-ec4d-45f0-9599-dd1ea95f53bb)

(2)Open Custom Blueprint System/WBP/WBP_BPNode. You need to modify these two nodes.

![image](https://github.com/user-attachments/assets/7bb89f91-bca3-4874-9219-ad972fbcb579)

(3)Open PreConstract. Customize the text, input box, checkbox, and pin logic of BPNode here.

![image](https://github.com/user-attachments/assets/e9c4b38f-4ed5-4656-86be-2059534bfdae)

(4)Open Node operation logic. Modify the running logic of BPNode here.

![image](https://github.com/user-attachments/assets/9cc0ad70-7d4f-4e56-96d7-8f31ac31b6c0)

(5)Modify the output pin logic of BPNode here.

![image](https://github.com/user-attachments/assets/dceee181-78bf-445c-90fe-0dd45fe8c452)

(6)Open Custom Blueprint System/WBP/WBP_EventsGraph. Selective modification of the content in the image.

![image](https://github.com/user-attachments/assets/4f26979e-1b5d-4ecb-b7dc-afc34405049e)

3.How to change the background image of the Events Graph?

Open Custom Blueprint System/WBP/WBP_EventsGraph. You just need to modify the Brush of the Background Panel.

![image](https://github.com/user-attachments/assets/29b4e0f7-bc18-4a89-99a7-2ec05a9e93a7)

4.How to change the connections between nodes?

   Open Custom Blueprint System/WBP/BPML_BP. Modify Calculation Lines
   
![image](https://github.com/user-attachments/assets/0658a65a-16ca-4586-9311-f3d6b6fab5ce)

5.Extended application:
(1) You can use Calculate Muse Offset in your other UI, such as map. This MACROS can map the mouse viewport position to the UI panel that has been scaled and offset at different resolutions, in order to achieve smooth movement of the UI on the scaled and offset UI panel.
(2) You can design the Custom Blueprint System as a puzzle solving system to increase gameplay
(3) You can encapsulate some functions into BPNodes to allow players to implement in-game programming, greatly increasing gameplay
(4) You can use it in some visualization scenarios, etc
