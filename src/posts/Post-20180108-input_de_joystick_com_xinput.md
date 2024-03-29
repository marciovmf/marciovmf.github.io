{{"Input de joystick com XInput"}}
Uma das premissas que adotei para o desenvolvimento da minha game [engine](https://github.com/marciovmf/ldare) é usar ao máximo os recursos nativos de cada sistema e evitar dependências tanto para o programador, quanto para o jogador. E foi pesquisando soluções nativas para implementar suporte a joystick que encontrei o [XInput](https://msdn.microsoft.com/pt-br/library/windows/desktop/ee417014(v=vs.85).aspx), que veio substituir o antigo [DirectInput](https://msdn.microsoft.com/pt-br/library/windows/desktop/ee416842(v=vs.85).aspx) . Neste post, vou mostrar como utilizei o XINPUT para dar suporte a joystick na minha engine e mais importante, sem forçar uma dependência em tempo de execução.

## Múltiplas versões

Existem [3 versões](https://msdn.microsoft.com/pt-br/library/windows/desktop/hh405051(v=vs.85).aspx) do XINPUT, cada uma distribuída em uma versão diferente do windows.

   XInput 1.4:  Nativa no Windows 8 e obrigatória para Windows Store apps ou desktop apps para Windows8.
   XInput 9.1.0: XInput 9.1.0 nativa no Windows Vista, Windows 7, e Windows 8.
   XInput 1.3: XInput 1.3 é parte do [DirectX SDK](https://msdn.microsoft.com/en-us/library/windows/desktop/ee663274(v=vs.85).aspx) com suporte pra Windows Vista, Windows 7, e Windows 8.

## Decidindo que versão usar

Recapitulando, temos 3 versões possíveis – todas elas nativas para alguma versão do Windows. Escolher uma, entretanto, significa automaticamente não suportar as demais. Caso o jogo rode em um ambiente sem a versão escolhida, o jogador deveria instalar a versão correta do DirectX. Evidentemente, este cenário viola completamente a premissa que comentei há pouco.

Por isso, eu decidi não “_linkar_” estaticamente com nenhuma das versões e simplesmente tentar carregar – em tempo de execução – a dll do XInput que estiver disponível no sistema. Assim, desde que a engine exponha apenas funcionalidades básicas, comuns às 3 versões, é possível suportar uma quantidade maior de sistemas operacionais.

A inicialização do XInput ficou assim:

      void Win32_initXInput()
      {
        char* xInputDllName = "xinput1_1.dll"; 
        HMODULE hXInput = LoadLibraryA(xInputDllName);
        if (!hXInput)
        {				
          xInputDllName = "xinput9_1_0.dll";
          hXInput = LoadLibraryA(xInputDllName);
        }
      
        if (!hXInput)
        {
          xInputDllName = "xinput1_3.dll";
          hXInput = LoadLibraryA(xInputDllName);
        }
      
        if (!hXInput)
        {
          LogError("could not initialize xinput. No valid xinput dll found");
          XInputGetState = XInputGetStateDummy;
          XInputSetState = XInputGetStateDummy;		
          return;
        }
      
        LogInfo("Initializing xinput %s", xInputDllName);
      
        //get xinput function pointers
        XInputGetState = (XInputGetStateFunc*) 
          GetProcAddress(hXInput, "XInputGetState");
        XInputSetState = (XInputSetStateFunc*)
          GetProcAddress(hXInput, "XInputSetState");
      
        if (!XInputGetState) XInputGetState = XInputGetStateDummy;
        if (!XInputSetState) XInputSetState = XInputGetStateDummy;
      }


A função tenta carregar, da mais antiga para a mais recente, qualquer dll do XInput que esteja disponível no sistema. Uma vez carregada a dll, obtenho manualmente os ponteiros para as funções que preciso chamar. Especificamente XInputGetState e XinputSetState que me permitem interações básicas como vibrar e ler o estado dos botões.

## API de input da engine

Minha proposta para a API de input da engine segue a filosofia da [Unity3D](https://unity3d.com/pt), onde se pode checar não só se um botão está [pressionado ou não](https://docs.unity3d.com/ScriptReference/Input.GetButton.html), mas também se foi [pressionado](https://docs.unity3d.com/ScriptReference/Input.GetButtonDown.html) ou [liberado](https://docs.unity3d.com/ScriptReference/Input.GetButtonUp.html) durante o frame atual.

Os eixos analógicos são representados com um float de 32 bits que vai de -1 até 1. O estado dos botões é representado por um único byte, onde o bit 0 indica o estado do botão e o bit 1, indica se este botão mudou de estado no frame atual.

Abaixo, a interface da API de input de joystick.

      #define KEYSTATE_PRESSED  0x01	
      #define KEYSTATE_CHANGED  0x02
      
      typedef int8 KeyState;
      
      inline int8 getButton(uint16 key, uint16 index = 0) const;
      inline int8 getButtonDown(uint16 key, uint16 index = 0) const;
      inline int8 getButtonUp(uint16 key, uint16 index = 0) const;
      inline float getAxis(uint16 axis, uint16 index = 0) const;
      
      //digital buttons
      #define GAMEPAD_DPAD_UP	0x00
      #define GAMEPAD_DPAD_DOWN	0x01
      #define GAMEPAD_DPAD_LEFT	0x02
      #define GAMEPAD_DPAD_RIGHT	0x03
      #define GAMEPAD_START	0x04
      #define GAMEPAD_FN1	0x04
      #define GAMEPAD_BACK	0x05
      #define GAMEPAD_FN2	0x05
      #define GAMEPAD_LEFT_THUMB	0x06
      #define GAMEPAD_RIGHT_THUMB	0x07
      #define GAMEPAD_LEFT_SHOULDER	0x08
      #define GAMEPAD_RIGHT_SHOULDER	0x09
      #define GAMEPAD_A	0x0A
      #define GAMEPAD_BTN1	0x0A
      #define GAMEPAD_B	0x0B
      #define GAMEPAD_BTN2	0x0B
      #define GAMEPAD_X	0x0C
      #define GAMEPAD_BTN3	0x0C
      #define GAMEPAD_Y	0x0D
      #define GAMEPAD_BTN4	0x0D
      
      // axis
      #define GAMEPAD_AXIS_LX 0x00
      #define GAMEPAD_AXIS_LY 0x01
      #define GAMEPAD_AXIS_RX 0x02
      #define GAMEPAD_AXIS_RY 0x03
      #define GAMEPAD_AXIS_LTRIGGER 0x04
      #define GAMEPAD_AXIS_RTRIGGER 0x05


Uma vez estabelecida a interface, implementa-la com XINPUT foi mais simples do que imaginei. O XINPUT é bastante enxuto, em contraste com a maioria das outras APIs da Microsoft.

## Fazendo polling dos joysticks

Obter o estado do joystick a cada frame é tão simples quanto fazer uma chamada a [XInputGetState](https://msdn.microsoft.com/pt-br/library/windows/desktop/microsoft.directx_sdk.reference.xinputgetstate(v=vs.85).aspx) de onde se pode obter uma estrutura [XINPUT\_GAMEPAD](https://msdn.microsoft.com/pt-br/library/windows/desktop/microsoft.directx_sdk.reference.xinput_gamepad(v=vs.85).aspx).

A função retorna **ERROR\_DEVICE\_NOT\_CONNECTED** para joysticks desconectados.


      static inline void Win32_processGamepadInput(ldare::Input& gameInput)
      {
        // clear 'changed' bit from input key state
        for(int gamepadIndex=0; gamepadIndex < MAX_GAMEPADS ; gamepadIndex++)
        {
          for(int i=0; i < GAMEPAD_MAX_DIGITAL_BUTTONS ; i++)
          {
            gameInput.gamepad[gamepadIndex].button[i] &= ~KEYSTATE_CHANGED;
          }
        }
      
        // get gamepad input
        for(int16 gamepadIndex = 0; gamepadIndex < MAX_GAMEPADS; gamepadIndex++)
        {
          ldare::platform::XINPUT_STATE gamepadState;
          Gamepad& gamepad = gameInput.gamepad[gamepadIndex];
      
          // ignore unconnected controllers
          if ( platform::XInputGetState(gamepadIndex, &gamepadState) == ERROR_DEVICE_NOT_CONNECTED )
          {
            if ( gamepad.connected)
            {
              gamepad = {};					
            }
            gamepad.connected = 0;
            continue;
          }
          // Get digital buttons here
          // Get axis here
          // Get Triggers here
        }
        ...
      }


## Botões digitais

A estrutura  [XINPUT\_GAMEPAD](https://msdn.microsoft.com/pt-br/library/windows/desktop/microsoft.directx_sdk.reference.xinput_gamepad(v=vs.85).aspx) contém um array com o estado de cada botão digital do joystick, bastando checar o índice dobotão correspondente. Valores > 0 indicam que o botão está pressionado.

      // Get digital buttons here
        WORD buttons = gamepadState.Gamepad.wButtons;
        uint8 isDown=0;
        uint8 wasDown=0;
      
      #define GET_GAMEPAD_BUTTON(btn) do {\
        isDown = (buttons & XINPUT_##btn) > 0;\
        wasDown = gamepad.button[btn] & KEYSTATE_PRESSED;\
        gamepad.button[btn] = ((isDown != wasDown) << 0x01) | isDown;\
      } while(0)
      
        GET_GAMEPAD_BUTTON(GAMEPAD_DPAD_UP);			
        GET_GAMEPAD_BUTTON(GAMEPAD_DPAD_DOWN);
        GET_GAMEPAD_BUTTON(GAMEPAD_DPAD_LEFT);
        GET_GAMEPAD_BUTTON(GAMEPAD_DPAD_RIGHT);
        GET_GAMEPAD_BUTTON(GAMEPAD_START);
        GET_GAMEPAD_BUTTON(GAMEPAD_BACK);
        GET_GAMEPAD_BUTTON(GAMEPAD_LEFT_THUMB);
        GET_GAMEPAD_BUTTON(GAMEPAD_RIGHT_THUMB);
        GET_GAMEPAD_BUTTON(GAMEPAD_LEFT_SHOULDER);
        GET_GAMEPAD_BUTTON(GAMEPAD_RIGHT_SHOULDER);
        GET_GAMEPAD_BUTTON(GAMEPAD_A);
        GET_GAMEPAD_BUTTON(GAMEPAD_B);
        GET_GAMEPAD_BUTTON(GAMEPAD_X);
        GET_GAMEPAD_BUTTON(GAMEPAD_Y);
      
      #undef SET_GAMEPAD_BUTTON


## Eixos e deadzones

De maneira similar, os eixos são lidos da mesma estrutura. Os valores vão de -32768 até 32767, onde 0 indica que o eixo está centralizado. Note que é necessário considerar uma zona morta, out dead zone, onde as leituras devem ignoradas, pois é possível ler valores diferentes de 0 mesmo sem mover os eixos em nenhuma direção.

Após obter o estado dos eixos, a função lê o estado dos triggers, tem um intervalo de 0 a 255 e também considero uma dead zone para evitar falsos positivos, como no caso dos eixos.


      static inline void Win32_processGamepadInput(ldare::Input& gameInput)
      {
        ...
        //TODO: Make these calculations directly in assembly to make it faster
      #define GAMEPAD_AXIS_VALUE(value) (value/(float)(value < 0 ? XINPUT_MIN_AXIS_VALUE * -1: XINPUT_MAX_AXIS_VALUE))
      #define GAMEPAD_AXIS_IS_DEADZONE(value, deadzone) ( value > -deadzone && value < deadzone)
      
        // Left thumb axis
        int32 axisX = gamepadState.Gamepad.sThumbLX;
        int32 axisY = gamepadState.Gamepad.sThumbLY;
        int32 deadZone = XINPUT_GAMEPAD_LEFT_THUMB_DEADZONE;
      
        gamepad.axis[GAMEPAD_AXIS_LX] = GAMEPAD_AXIS_IS_DEADZONE(axisX, deadZone) ? 0.0f :
          GAMEPAD_AXIS_VALUE(axisX);
        gamepad.axis[GAMEPAD_AXIS_LY] = GAMEPAD_AXIS_IS_DEADZONE(axisY, deadZone) ? 0.0f :	
          GAMEPAD_AXIS_VALUE(axisY);
      
        // Right thumb axis
        axisX = gamepadState.Gamepad.sThumbRX;
        axisY = gamepadState.Gamepad.sThumbRY;
        deadZone = XINPUT_GAMEPAD_RIGHT_THUMB_DEADZONE;
        gamepad.axis[GAMEPAD_AXIS_RX] = GAMEPAD_AXIS_IS_DEADZONE(axisX, deadZone) ? 0.0f :
          GAMEPAD_AXIS_VALUE(axisX);
      
        gamepad.axis[GAMEPAD_AXIS_RY] = GAMEPAD_AXIS_IS_DEADZONE(axisY, deadZone) ? 0.0f :	
          GAMEPAD_AXIS_VALUE(axisY);
      
        // Left trigger
        axisX = gamepadState.Gamepad.bLeftTrigger;
        axisY = gamepadState.Gamepad.bRightTrigger;
        deadZone = XINPUT_GAMEPAD_TRIGGER_THRESHOLD;
      
        gamepad.axis[GAMEPAD_AXIS_LTRIGGER] = GAMEPAD_AXIS_IS_DEADZONE(axisX, deadZone) ? 0.0f :	
          axisX/(float) XINPUT_MAX_TRIGGER_VALUE;
      
        gamepad.axis[GAMEPAD_AXIS_RTRIGGER] = GAMEPAD_AXIS_IS_DEADZONE(axisY, deadZone) ? 0.0f :	
          axisY/(float) XINPUT_MAX_TRIGGER_VALUE;
      
      #undef GAMEPAD_AXIS_IS_DEADZONE
      #undef GAMEPAD_AXIS_VALUE
        gamepad.connected = 1;
      
      }


## Considerações finais

Ainda não tive oportunidade nem necessidade de avaliar latência ou precisão das leituras, mas é sem dúvida algo que precisarei fazer no futuro.

No mais, eu achei o XINPUT bastante enxuto e o fato de ser nativo torna ainda mais interessante. Não surpreende ser tão utilizada por [inúmeros jogos](http://www.self.gutenberg.org/articles/eng/List_of_games_with_XInput_support) pra PC atualmente. Me pergunto se terei a mesma facilidade com a [API de Joystick da apple](https://developer.apple.com/library/content/documentation/ServicesDiscovery/Conceptual/GameControllerPG/Introduction/Introduction.html) quando for implementar isso no Mac.

O código com toda a implementação pode ser acessado no [repo do projeto](https://github.com/marciovmf/ldare) no github.

Happy Coding.
