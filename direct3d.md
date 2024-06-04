###  HelloWindow -> HelloTriangle로 추가되는 내용

D3D12HelloWindow 클래스는 기본적인 DirectX 12 윈도우 초기화와 렌더링 루프를 포함하고 있으며,
D3D12HelloTriangle 클래스는 삼각형을 렌더링하기 위한 추가적인 초기화 및 자산 로딩을 포함합니다. 두 클래스의 주요 차이점은 다음과 같습니다.

#### 1. Root Signature 및 Pipeline State 객체 생성

D3D12HelloTriangle에서는 루트 서명과 그래픽 파이프라인 상태 객체(PSO)를 생성합니다. 이는 셰이더와 렌더링 상태를 정의하는 중요한 단계입니다.

```
// Create an empty root signature.
{
    CD3DX12_ROOT_SIGNATURE_DESC rootSignatureDesc;
    rootSignatureDesc.Init(0, nullptr, 0, nullptr, D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT);

    ComPtr<ID3DBlob> signature;
    ComPtr<ID3DBlob> error;
    ThrowIfFailed(D3D12SerializeRootSignature(&rootSignatureDesc, D3D_ROOT_SIGNATURE_VERSION_1, &signature, &error));
    ThrowIfFailed(m_device->CreateRootSignature(0, signature->GetBufferPointer(), signature->GetBufferSize(), IID_PPV_ARGS(&m_rootSignature)));
}

// Create the pipeline state, which includes compiling and loading shaders.
{
    ComPtr<ID3DBlob> vertexShader;
    ComPtr<ID3DBlob> pixelShader;

    // Shader 컴파일
    ThrowIfFailed(D3DCompileFromFile(GetAssetFullPath(L"shaders.hlsl").c_str(), nullptr, nullptr, "VSMain", "vs_5_0", compileFlags, 0, &vertexShader, nullptr));
    ThrowIfFailed(D3DCompileFromFile(GetAssetFullPath(L"shaders.hlsl").c_str(), nullptr, nullptr, "PSMain", "ps_5_0", compileFlags, 0, &pixelShader, nullptr));

    // 입력 레이아웃 정의
    D3D12_INPUT_ELEMENT_DESC inputElementDescs[] =
    {
        { "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 0, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
        { "COLOR", 0, DXGI_FORMAT_R32G32B32A32_FLOAT, 0, 12, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 }
    };

    // PSO 기술
    D3D12_GRAPHICS_PIPELINE_STATE_DESC psoDesc = {};
    psoDesc.InputLayout = { inputElementDescs, _countof(inputElementDescs) };
    psoDesc.pRootSignature = m_rootSignature.Get();
    psoDesc.VS = CD3DX12_SHADER_BYTECODE(vertexShader.Get());
    psoDesc.PS = CD3DX12_SHADER_BYTECODE(pixelShader.Get());
    psoDesc.RasterizerState = CD3DX12_RASTERIZER_DESC(D3D12_DEFAULT);
    psoDesc.BlendState = CD3DX12_BLEND_DESC(D3D12_DEFAULT);
    psoDesc.DepthStencilState.DepthEnable = FALSE;
    psoDesc.DepthStencilState.StencilEnable = FALSE;
    psoDesc.SampleMask = UINT_MAX;
    psoDesc.PrimitiveTopologyType = D3D12_PRIMITIVE_TOPOLOGY_TYPE_TRIANGLE;
    psoDesc.NumRenderTargets = 1;
    psoDesc.RTVFormats[0] = DXGI_FORMAT_R8G8B8A8_UNORM;
    psoDesc.SampleDesc.Count = 1;
    ThrowIfFailed(m_device->CreateGraphicsPipelineState(&psoDesc, IID_PPV_ARGS(&m_pipelineState)));
}
```

#### 2. 버텍스 버퍼 생성

삼각형을 정의하는 버텍스 데이터를 생성하고 이를 GPU로 전송하기 위한 버텍스 버퍼를 만듭니다.

```
{
    // 삼각형의 정점을 정의합니다.
    Vertex triangleVertices[] =
    {
        { { 0.0f, 0.25f * m_aspectRatio, 0.0f }, { 1.0f, 0.0f, 0.0f, 1.0f } },
        { { 0.25f, -0.25f * m_aspectRatio, 0.0f }, { 0.0f, 1.0f, 0.0f, 1.0f } },
        { { -0.25f, -0.25f * m_aspectRatio, 0.0f }, { 0.0f, 0.0f, 1.0f, 1.0f } }
    };

    const UINT vertexBufferSize = sizeof(triangleVertices);

    // 업로드 힙을 사용하여 버텍스 버퍼를 생성합니다.
    ThrowIfFailed(m_device->CreateCommittedResource(
        &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_UPLOAD),
        D3D12_HEAP_FLAG_NONE,
        &CD3DX12_RESOURCE_DESC::Buffer(vertexBufferSize),
        D3D12_RESOURCE_STATE_GENERIC_READ,
        nullptr,
        IID_PPV_ARGS(&m_vertexBuffer)));

    // 버텍스 데이터를 버퍼에 복사합니다.
    UINT8* pVertexDataBegin;
    CD3DX12_RANGE readRange(0, 0); // CPU에서 읽지 않을 예정입니다.
    ThrowIfFailed(m_vertexBuffer->Map(0, &readRange, reinterpret_cast<void**>(&pVertexDataBegin)));
    memcpy(pVertexDataBegin, triangleVertices, sizeof(triangleVertices));
    m_vertexBuffer->Unmap(0, nullptr);

    // 버텍스 버퍼 뷰를 초기화합니다.
    m_vertexBufferView.BufferLocation = m_vertexBuffer->GetGPUVirtualAddress();
    m_vertexBufferView.StrideInBytes = sizeof(Vertex);
    m_vertexBufferView.SizeInBytes = vertexBufferSize;
}
```

#### 3. 명령 리스트 업데이트 및 렌더링

D3D12HelloTriangle에서는 명령 리스트를 업데이트하여 삼각형을 렌더링합니다. 이는 D3D12HelloWindow의 기본 렌더링 루프에 추가됩니다.

```
void D3D12HelloTriangle::PopulateCommandList()
{
    // 명령 리스트 할당자를 리셋합니다.
    ThrowIfFailed(m_commandAllocator->Reset());

    // 명령 리스트를 리셋합니다.
    ThrowIfFailed(m_commandList->Reset(m_commandAllocator.Get(), m_pipelineState.Get()));

    // 필요한 상태를 설정합니다.
    m_commandList->SetGraphicsRootSignature(m_rootSignature.Get());
    m_commandList->RSSetViewports(1, &m_viewport);
    m_commandList->RSSetScissorRects(1, &m_scissorRect);

    // 백 버퍼를 렌더 타겟으로 사용한다고 표시합니다.
    m_commandList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(m_renderTargets[m_frameIndex].Get(), D3D12_RESOURCE_STATE_PRESENT, D3D12_RESOURCE_STATE_RENDER_TARGET));

    CD3DX12_CPU_DESCRIPTOR_HANDLE rtvHandle(m_rtvHeap->GetCPUDescriptorHandleForHeapStart(), m_frameIndex, m_rtvDescriptorSize);
    m_commandList->OMSetRenderTargets(1, &rtvHandle, FALSE, nullptr);

    // 명령을 기록합니다.
    const float clearColor[] = { 0.0f, 0.2f, 0.4f, 1.0f };
    m_commandList->ClearRenderTargetView(rtvHandle, clearColor, 0, nullptr);
    m_commandList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
    m_commandList->IASetVertexBuffers(0, 1, &m_vertexBufferView);
    m_commandList->DrawInstanced(3, 1, 0, 0);

    // 백 버퍼를 프레젠트로 사용한다고 표시합니다.
    m_commandList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(m_renderTargets[m_frameIndex].Get(), D3D12_RESOURCE_STATE_RENDER_TARGET, D3D12_RESOURCE_STATE_PRESENT));

    ThrowIfFailed(m_commandList->Close());
}
```
위 내용을 정리하면 
1. 정점 데이터 정의 및 버퍼 생성 : 삼각형을 정의하는 정점 데이터를 생성하고, 이를 GPU 메모리에 저장하기 위한 버퍼를 생성합니다.
2. 루트 서명 및 파이프라인 상태 객체(PSO)설정: 셰이더와 파이프라인 상태 객체를 정의하고 설정하여 GPU에서 렌더링 파이프라인을 구성합니다.
3. 명령 목록에 렌더링 명령 기록: 렌더링 명령을 명령 목록에 기록하여 GPU에서 삼각형을 실제로 그릴 수 있오록 합니다.

이와 같은 변경 사항을 통해 D3D12HelloTriangle 클래스는 삼각형을 렌더링할 수 있게 됩니다. 주요 추가 항목은 루트 서명, 파이프라인 상태 객체, 버텍스 버퍼 생성 및 명령 리스트 업데이트입니다.

### HelloTriangle -> HelloTexture로 추가된 내용

















### 실습화면

![image](https://github.com/qkrgudals1030/direct3d/assets/50895124/0a6dc12e-959c-44f0-ac62-1c5b4de084b4)


![image](https://github.com/qkrgudals1030/direct3d/assets/50895124/31d6d707-50f6-4ce5-91ad-9800525bde04)


![image](https://github.com/qkrgudals1030/direct3d/assets/50895124/9a1cc1ad-89c2-4edb-8bfa-53c863e6a042)












