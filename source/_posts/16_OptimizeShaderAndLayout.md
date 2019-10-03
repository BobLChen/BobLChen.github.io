---
title: 16_OptimizeShaderAndLayout
date: 2019-08-27 22:49:01
tags:
- Vulkan
- 物理设备
- 3D
- Demo
categories:
- Vulkan
---

[16_OptimizeShaderAndLayout](https://github.com/BobLChen/VulkanDemos/tree/master/examples/16_OptimizeShaderAndLayout)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

在前面的几个Demo里面，我们在频繁的创建**DescriptorLayout**，到目前为此我已经对这个事情比较厌烦，在后续的Demo里面，我们不可能为了一些特效写个几百行的DescriptorLayout的创建代码，而一旦Shader发生了变化又要回来来改这些代码。我觉得因为Shader里面的Layout发生了改变而导致我要来改Demo的代码，这个事情会显得比较蠢。在这个Demo里面，我们势必要基本解决这个问题，即要能够根据Shader自动产生出DescriptorLayout出来。

<!-- more -->

## 16_OptimizeShaderAndLayout

目前我采样的方式比较直接粗暴，但是我希望可以通过这种方式来启发大家采样更加稳健的方案。在这个Demo里面，我采用了一个SPIRV反编译器来反编译我的二进制Shader，通过反编译器获得相关的DescriptorLayout信息。在真实的生成环境中，我建议大家利用SPIRV编译器在Editor中对Shader进行编译，然后获得相关的信息，最后存储到Shader中，最后加载Shader数据通过一次反序列化获得所有的数据。

SPIRV反编译库用到了**SPIRV_CROSS**这个库，我已经通过CMake集成到了Demo里面。下面我们来设想一下自动产生DescriptorLayout的流程。

我们新建一个**DVKShader.h**和**DVKShader.cpp**用于存放相关的代码。

### 产生流程

#### 1、加载Shader二进制数据
这个不用多说，我们肯定要载入对应的二进制数据，才能完成后续的功能。目前我们知道的Shader类型有以下几种：
- Vertex shader
- Geometry shdaer
- Tessellation control shader
- Tessellation evaluation shader
- Compute shader
- Fragment shader
目前只有这几种Shader，但是新出的**RTX**系列增加了**Mesh Shader**，**Mesh Shader**我们暂时不考虑。

#### 2、创建ShaderModule
在Vulkan里面，Pipeline需要多种Shader配合使用，每一种Shader对应一个Shader Module。在这个Demo系列里面，我们把**多种Shader Module**的组合当成一个**Shader**。但是我们任然需要对Shader Module进行一个简单的封装。代码如下：
```c++
class DVKShaderModule
{
private:
    DVKShaderModule()
    {

    }

public:
    
    ~DVKShaderModule()
    {
        if (handle != VK_NULL_HANDLE) 
        {
            vkDestroyShaderModule(device, handle, VULKAN_CPU_ALLOCATOR);
            handle = VK_NULL_HANDLE;
        }
        
        if (data) 
        {
            delete[] data;
            data = nullptr;
        }
    }

    static DVKShaderModule* Create(std::shared_ptr<VulkanDevice> vulkanDevice, const char* filename, VkShaderStageFlagBits stage);
    
public:

    VkDevice				device;
    VkShaderStageFlagBits	stage;
    VkShaderModule			handle;
    uint8*					data;
    uint32					size;
};

DVKShaderModule* DVKShaderModule::Create(std::shared_ptr<VulkanDevice> vulkanDevice, const char* filename, VkShaderStageFlagBits stage)
{
    VkDevice device = vulkanDevice->GetInstanceHandle();

    uint8* dataPtr  = nullptr;
    uint32 dataSize = 0;
    if (!FileManager::ReadFile(filename, dataPtr, dataSize))
    {
        MLOGE("Failed load file:%s", filename);
        return nullptr;
    }
    
    VkShaderModuleCreateInfo moduleCreateInfo;
    ZeroVulkanStruct(moduleCreateInfo, VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO);
    moduleCreateInfo.codeSize = dataSize;
    moduleCreateInfo.pCode    = (uint32_t*)dataPtr;
    
    VkShaderModule shaderModule = VK_NULL_HANDLE;
    VERIFYVULKANRESULT(vkCreateShaderModule(device, &moduleCreateInfo, VULKAN_CPU_ALLOCATOR, &shaderModule));

    DVKShaderModule* dvkModule = new DVKShaderModule();
    dvkModule->data   = dataPtr;
    dvkModule->size   = dataSize;
    dvkModule->device = device;
    dvkModule->handle = shaderModule;
    dvkModule->stage  = stage;

    return dvkModule;
}

```
通过**DVKShaderModule**我们可以获取到对应的二进制数据、ShaderModule、ShaderStage。

#### 4、解析DescriptorLayout
在这个Demo里面，我将解析过程称为编译，编译的时候就是对每个Shader Module进行一次解析，获取到对应的Layout信息。代码如下：
```c++
DVKShader* DVKShader::Create(std::shared_ptr<VulkanDevice> vulkanDevice, bool dynamicUBO, const char* vert, const char* frag, const char* geom, const char* comp, const char* tesc, const char* tese)
{
    DVKShaderModule* vertModule = vert ? DVKShaderModule::Create(vulkanDevice, vert, VK_SHADER_STAGE_VERTEX_BIT)   : nullptr;
    DVKShaderModule* fragModule = frag ? DVKShaderModule::Create(vulkanDevice, frag, VK_SHADER_STAGE_FRAGMENT_BIT) : nullptr;
    DVKShaderModule* geomModule = geom ? DVKShaderModule::Create(vulkanDevice, geom, VK_SHADER_STAGE_GEOMETRY_BIT) : nullptr;
    DVKShaderModule* compModule = comp ? DVKShaderModule::Create(vulkanDevice, comp, VK_SHADER_STAGE_COMPUTE_BIT) : nullptr;
    DVKShaderModule* tescModule = tesc ? DVKShaderModule::Create(vulkanDevice, tesc, VK_SHADER_STAGE_TESSELLATION_CONTROL_BIT)    : nullptr;
    DVKShaderModule* teseModule = tese ? DVKShaderModule::Create(vulkanDevice, tese, VK_SHADER_STAGE_TESSELLATION_EVALUATION_BIT) : nullptr;
    
    DVKShader* shader = new DVKShader();
    shader->device     = vulkanDevice->GetInstanceHandle();
    shader->dynamicUBO = dynamicUBO;
    
    shader->vertShaderModule = vertModule;
    shader->fragShaderModule = fragModule;
    shader->geomShaderModule = geomModule;
    shader->compShaderModule = compModule;
    shader->tescShaderModule = tescModule;
    shader->teseShaderModule = teseModule;
    
    shader->Compile();
    
    return shader;
}

void DVKShader::Compile()
{
    ProcessShaderModule(vertShaderModule);
    ProcessShaderModule(fragShaderModule);
    ProcessShaderModule(geomShaderModule);
    ProcessShaderModule(compShaderModule);
    ProcessShaderModule(tescShaderModule);
    ProcessShaderModule(teseShaderModule);
    GenerateInputInfo();
    GenerateLayout();
}

```

**Compile**函数主要就是在解析每个ShaderModule，然后获取Vertex Shader的输入信息，产生Layout信息。接下来我们着重看一下**ProcessShaderModule**函数，代码如下：

```c++
void DVKShader::ProcessShaderModule(DVKShaderModule* shaderModule)
{
    if (!shaderModule) {
        return;
    }

    // 保存StageInfo
    VkPipelineShaderStageCreateInfo shaderCreateInfo;
    ZeroVulkanStruct(shaderCreateInfo, VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO);
    shaderCreateInfo.stage  = shaderModule->stage;
    shaderCreateInfo.module = shaderModule->handle;
    shaderCreateInfo.pName  = "main";
    shaderStageCreateInfos.push_back(shaderCreateInfo);

    // 反编译Shader获取相关信息
    spirv_cross::Compiler compiler((uint32*)shaderModule->data, shaderModule->size / sizeof(uint32));
    spirv_cross::ShaderResources resources = compiler.get_shader_resources();
    
    ProcessUniformBuffers(compiler, resources, shaderModule->stage);
    ProcessTextures(compiler, resources, shaderModule->stage);
    ProcessInput(compiler, resources, shaderModule->stage);
}

```

**spirv_cross**就是我们使用到的反编译器，我们传入Shader对应的二进制数据，然后通过**spirv_cross::Compiler**反编译ShaderModule，获取到反编译之后的数据。
```c++
spirv_cross::Compiler compiler((uint32*)shaderModule->data, shaderModule->size / sizeof(uint32));
spirv_cross::ShaderResources resources = compiler.get_shader_resources();
```

反编译完成之后，我们目前掌握的Demo知识里面，只有Uniform、Texture以及Input数据。因此我们针对这三个类型进行解析存储。
```c++
void DVKShader::ProcessUniformBuffers(spirv_cross::Compiler& compiler, spirv_cross::ShaderResources& resources, VkShaderStageFlags stageFlags)
{
    // 获取Uniform Buffer信息
    for (int32 i = 0; i < resources.uniform_buffers.size(); ++i)
    {
        spirv_cross::Resource& res      = resources.uniform_buffers[i];
        spirv_cross::SPIRType type      = compiler.get_type(res.type_id);
        spirv_cross::SPIRType base_type = compiler.get_type(res.base_type_id);
        const std::string &varName      = compiler.get_name(res.id);
        const std::string &typeName     = compiler.get_name(res.base_type_id);
        uint32 uniformBufferStructSize  = compiler.get_declared_struct_size(type);
        
        int32 set     = compiler.get_decoration(res.id, spv::DecorationDescriptorSet);
        int32 binding = compiler.get_decoration(res.id, spv::DecorationBinding);
        
        // [layout (binding = 0) uniform MVPDynamicBlock] 标记为Dynamic的buffer
        VkDescriptorSetLayoutBinding setLayoutBinding = {};
        setLayoutBinding.binding             = binding;
        setLayoutBinding.descriptorType     = (typeName.find("Dynamic") != std::string::npos || dynamicUBO) ? VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC : VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
        setLayoutBinding.descriptorCount    = 1;
        setLayoutBinding.stageFlags         = stageFlags;
        setLayoutBinding.pImmutableSamplers = nullptr;
        
        setLayoutsInfo.AddDescriptorSetLayoutBinding(varName, set, setLayoutBinding);
        
        // 保存UBO变量信息
        auto it = bufferParams.find(varName);
        if (it == bufferParams.end())
        {
            BufferInfo bufferInfo = {};
            bufferInfo.set            = set;
            bufferInfo.binding        = binding;
            bufferInfo.bufferSize     = uniformBufferStructSize;
            bufferInfo.stageFlags     = stageFlags;
            bufferInfo.descriptorType = setLayoutBinding.descriptorType;
            bufferParams.insert(std::make_pair(varName, bufferInfo));
        }
        else
        {
            it->second.stageFlags |= setLayoutBinding.stageFlags;
        }
    }
}

void DVKShader::ProcessTextures(spirv_cross::Compiler& compiler, spirv_cross::ShaderResources& resources, VkShaderStageFlags stageFlags)
{
    // 获取Texture
    for (int32 i = 0; i < resources.sampled_images.size(); ++i)
    {
        spirv_cross::Resource& res      = resources.sampled_images[i];
        spirv_cross::SPIRType type      = compiler.get_type(res.type_id);
        spirv_cross::SPIRType base_type = compiler.get_type(res.base_type_id);
        const std::string&      varName = compiler.get_name(res.id);
        
        int32 set     = compiler.get_decoration(res.id, spv::DecorationDescriptorSet);
        int32 binding = compiler.get_decoration(res.id, spv::DecorationBinding);
        
        VkDescriptorSetLayoutBinding setLayoutBinding = {};
        setLayoutBinding.binding             = binding;
        setLayoutBinding.descriptorType     = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
        setLayoutBinding.descriptorCount    = 1;
        setLayoutBinding.stageFlags         = stageFlags;
        setLayoutBinding.pImmutableSamplers = nullptr;
        
        setLayoutsInfo.AddDescriptorSetLayoutBinding(varName, set, setLayoutBinding);
        
        auto it = imageParams.find(varName);
        if (it == imageParams.end())
        {
            ImageInfo imageInfo = {};
            imageInfo.set            = set;
            imageInfo.binding        = binding;
            imageInfo.stageFlags     = stageFlags;
            imageInfo.descriptorType = setLayoutBinding.descriptorType;
            imageParams.insert(std::make_pair(varName, imageInfo));
        }
        else
        {
            it->second.stageFlags |= stageFlags;
        }
    }
}

void DVKShader::ProcessInput(spirv_cross::Compiler& compiler, spirv_cross::ShaderResources& resources, VkShaderStageFlags stageFlags)
{
    if (stageFlags != VK_SHADER_STAGE_VERTEX_BIT) {
        return;
    }

    // 获取input信息
    for (int32 i = 0; i < resources.stage_inputs.size(); ++i)
    {
        spirv_cross::Resource& res = resources.stage_inputs[i];
        spirv_cross::SPIRType type = compiler.get_type(res.type_id);
        const std::string &varName = compiler.get_name(res.id);
        int32 inputAttributeSize   = type.vecsize;
        
        VertexAttribute attribute  = StringToVertexAttribute(varName.c_str());
        if (attribute == VertexAttribute::VA_None)
        {
            if (inputAttributeSize == 1) {
                attribute = VertexAttribute::VA_InstanceFloat1;
            }
            else if (inputAttributeSize == 2) {
                attribute = VertexAttribute::VA_InstanceFloat2;
            }
            else if (inputAttributeSize == 3) {
                attribute = VertexAttribute::VA_InstanceFloat3;
            }
            else if (inputAttributeSize == 4) {
                attribute = VertexAttribute::VA_InstanceFloat4;
            }
            MLOG("Not found attribute : %s, treat as instance attribute : %d.", varName.c_str(), int32(attribute));
        }
        
        // location必须连续
        int32 location = compiler.get_decoration(res.id, spv::DecorationLocation);
        DVKAttribute dvkAttribute = {};
        dvkAttribute.location  = location;
        dvkAttribute.attribute = attribute;
        m_InputAttributes.push_back(dvkAttribute);
    }
}

```

需要注意的一个地方是，不同的ShaderModule可以共享数据，即可以共享Layout，例如一个Uniform数据即可以被Vertex Shader使用，也可以被Fragment Shader使用，只需要保证他们的Layout一致，就可以避免创建多份Uniform数据。

#### 5、Vertex Input
Vertex Shader是包含了Input的Layout的，我们解析完Vertex Shader之后既可以产生出对应的Input Layout数据。目前我们对Input的数据处理比较粗暴，只设计了一个Buffer用来存储所有的数据，对顺序和格式也有特别的要求，这种方式自由度不高，但是目前我们Demo是足够使用。
```c++
void DVKShader::GenerateInputInfo()
{
    // 对inputAttributes进行排序，获取Attributes列表
    std::sort(m_InputAttributes.begin(), m_InputAttributes.end(), [](const DVKAttribute& a, const DVKAttribute& b) -> bool {
        return a.location < b.location;
    });
    
    // 对inputAttributes进行归类整理
    for (int32 i = 0; i < m_InputAttributes.size(); ++i)
    {
        VertexAttribute attribute = m_InputAttributes[i].attribute;
        perVertexAttributes.push_back(attribute);
    }
    
    // 生成Bindinfo
    inputBindings.resize(0);
    if (perVertexAttributes.size() > 0)
    {
        int32 stride = 0;
        for (int32 i = 0; i < perVertexAttributes.size(); ++i) {
            stride += VertexAttributeToSize(perVertexAttributes[i]);
        }
        VkVertexInputBindingDescription perVertexInputBinding = {};
        perVertexInputBinding.binding   = 0;
        perVertexInputBinding.stride    = stride;
        perVertexInputBinding.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;
        inputBindings.push_back(perVertexInputBinding);
    }
    
    // 生成attributes info
    int location = 0;
    if (perVertexAttributes.size() > 0)
    {
        int32 offset = 0;
        for (int32 i = 0; i < perVertexAttributes.size(); ++i)
        {
            VkVertexInputAttributeDescription inputAttribute = {};
            inputAttribute.binding  = 0;
            inputAttribute.location = location;
            inputAttribute.format   = VertexAttributeToVkFormat(perVertexAttributes[i]);
            inputAttribute.offset   = offset;
            offset += VertexAttributeToSize(perVertexAttributes[i]);
            inputAttributes.push_back(inputAttribute);
            
            location += 1;
        }
    }
}
```

#### 6、生成Layout
根据之前反编译收集到的信息，我们完全可以产生出正确的**DescriptorSetLayout**信息出来。结合之前的Demo里面提到的，**DescriptorSetLayout**是可以多个的，主要可能是为了功能分类。在这里一步到位，把多Set的情况也支持上。
```c++
void DVKShader::GenerateLayout()
{
    std::vector<DVKDescriptorSetLayoutInfo>& setLayouts = setLayoutsInfo.setLayouts;
    
    // 先按照set进行排序
    std::sort(setLayouts.begin(), setLayouts.end(), [](const DVKDescriptorSetLayoutInfo& a, const DVKDescriptorSetLayoutInfo& b) -> bool {
        return a.set < b.set;
    });
    
    // 再按照binding进行排序
    for (int32 i = 0; i < setLayouts.size(); ++i)
    {
        std::vector<VkDescriptorSetLayoutBinding>& bindings = setLayouts[i].bindings;
        std::sort(bindings.begin(), bindings.end(), [](const VkDescriptorSetLayoutBinding& a, const VkDescriptorSetLayoutBinding& b) -> bool {
            return a.binding < b.binding;
        });
    }
    
    for (int32 i = 0; i < setLayoutsInfo.setLayouts.size(); ++i)
    {
        VkDescriptorSetLayout descriptorSetLayout = VK_NULL_HANDLE;
        DVKDescriptorSetLayoutInfo& setLayoutInfo = setLayoutsInfo.setLayouts[i];
        
        VkDescriptorSetLayoutCreateInfo descSetLayoutInfo;
        ZeroVulkanStruct(descSetLayoutInfo, VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO);
        descSetLayoutInfo.bindingCount = setLayoutInfo.bindings.size();
        descSetLayoutInfo.pBindings    = setLayoutInfo.bindings.data();
        VERIFYVULKANRESULT(vkCreateDescriptorSetLayout(device, &descSetLayoutInfo, VULKAN_CPU_ALLOCATOR, &descriptorSetLayout));

        descriptorSetLayouts.push_back(descriptorSetLayout);
    }

    VkPipelineLayoutCreateInfo pipeLayoutInfo;
    ZeroVulkanStruct(pipeLayoutInfo, VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO);
    pipeLayoutInfo.setLayoutCount = descriptorSetLayouts.size();
    pipeLayoutInfo.pSetLayouts    = descriptorSetLayouts.data();
    VERIFYVULKANRESULT(vkCreatePipelineLayout(device, &pipeLayoutInfo, VULKAN_CPU_ALLOCATOR, &pipelineLayout));
}
```

#### DVKDescriptorSetPool

之前的Demo里面也了解到**DescriptorSet**是资源与Shader的桥梁，DescriptorSet通过DescriptorLayout创建。既然我们在DVKShader里面有了**DescriptorLayout**，那么我们创建**DescriptorSet**也就轻而易举了。

在生产环境中，我们一般会制作出Shader，然后制作出非常非常多的Material，这些Material都是用到的同一个Shader，只是参数不同而已，例如Texture不同。那么反应到我们现在的系统里面，也就是**DescriptorSet**不同。由此可见，我们的**DVKShader**需要有能力产生出非常多的**DescriptorSet**，这些**DescriptorSet**负责沟通不同的资源。

为了更好的创建维护**DescriptorSet**，在这个Demo里面，我设计了一个简单的**DVKDescriptorSetPool**，用来维护这些**DescriptorSet**。代码如下：
```c++
class DVKDescriptorSetPool
{
public:
    DVKDescriptorSetPool(VkDevice inDevice, int32 inMaxSet, const DVKDescriptorSetLayoutsInfo& setLayoutsInfo, const std::vector<VkDescriptorSetLayout>& inDescriptorSetLayouts)
    {
        device  = inDevice;
        maxSet  = inMaxSet;
        usedSet = 0;
        descriptorSetLayouts = inDescriptorSetLayouts;

        std::vector<VkDescriptorPoolSize> poolSizes;
        for (int32 i = 0; i < setLayoutsInfo.setLayouts.size(); ++i)
        {
            const DVKDescriptorSetLayoutInfo& setLayoutInfo = setLayoutsInfo.setLayouts[i];
            for (int32 j = 0; j < setLayoutInfo.bindings.size(); ++j)
            {
                VkDescriptorPoolSize poolSize = {};
                poolSize.type            = setLayoutInfo.bindings[j].descriptorType;
                poolSize.descriptorCount = setLayoutInfo.bindings[j].descriptorCount;
                poolSizes.push_back(poolSize);
            }
        }

        VkDescriptorPoolCreateInfo descriptorPoolInfo;
        ZeroVulkanStruct(descriptorPoolInfo, VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO);
        descriptorPoolInfo.poolSizeCount = poolSizes.size();
        descriptorPoolInfo.pPoolSizes    = poolSizes.data();
        descriptorPoolInfo.maxSets       = maxSet;
        VERIFYVULKANRESULT(vkCreateDescriptorPool(inDevice, &descriptorPoolInfo, VULKAN_CPU_ALLOCATOR, &descriptorPool));
    }

    ~DVKDescriptorSetPool()
    {
        if (descriptorPool != VK_NULL_HANDLE)
        {
            vkDestroyDescriptorPool(device, descriptorPool, VULKAN_CPU_ALLOCATOR);
            descriptorPool = VK_NULL_HANDLE;
        }
    }

    bool IsFull()
    {
        return usedSet >= maxSet;
    }

    bool AllocateDescriptorSet(VkDescriptorSet* descriptorSet)
    {
        if (usedSet + descriptorSetLayouts.size() >= maxSet) {
            return false;
        }

        usedSet += descriptorSetLayouts.size();

        VkDescriptorSetAllocateInfo allocInfo;
        ZeroVulkanStruct(allocInfo, VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO);
        allocInfo.descriptorPool     = descriptorPool;
        allocInfo.descriptorSetCount = descriptorSetLayouts.size();
        allocInfo.pSetLayouts        = descriptorSetLayouts.data();
        VERIFYVULKANRESULT(vkAllocateDescriptorSets(device, &allocInfo, descriptorSet));

        return true;
    }
    
public:
    int32								maxSet;
    int32								usedSet;
    VkDevice							device = VK_NULL_HANDLE;
    std::vector<VkDescriptorSetLayout>	descriptorSetLayouts;
    VkDescriptorPool					descriptorPool = VK_NULL_HANDLE;
};

```

#### DVKDescriptorSet

有了**DescriptorSetPool**之后，我们再对**DescriptorSet**进行一下简单的封装，让它能够快速方便的对UniformBuffer和Texture进行Update操作。在前面通过反编译器获取了所有的Layout信息，通过Layout信息我们就可以查询**变量名称**对应的Layout类型，然后进行相关的Update操作。

```c++
class DVKDescriptorSet
{
public:

    DVKDescriptorSet()
    {

    }

    ~DVKDescriptorSet()
    {
        
    }
    
    void WriteImage(const std::string& name, DVKTexture* texture)
    {
        auto it = setLayoutsInfo.paramsMap.find(name);
        if (it == setLayoutsInfo.paramsMap.end()) 
        {
            MLOGE("Failed write buffer, %s not found!", name.c_str());
            return;
        }

        auto bindInfo = it->second;

        VkWriteDescriptorSet writeDescriptorSet;
        ZeroVulkanStruct(writeDescriptorSet, VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET);
        writeDescriptorSet.dstSet          = descriptorSets[bindInfo.set];
        writeDescriptorSet.descriptorCount = 1;
        writeDescriptorSet.descriptorType  = setLayoutsInfo.GetDescriptorType(bindInfo.set, bindInfo.binding);
        writeDescriptorSet.pBufferInfo     = nullptr;
        writeDescriptorSet.pImageInfo      = &(texture->descriptorInfo);
        writeDescriptorSet.dstBinding      = bindInfo.binding;
        vkUpdateDescriptorSets(device, 1, &writeDescriptorSet, 0, nullptr);
    }

    void WriteBuffer(const std::string& name, const VkDescriptorBufferInfo* bufferInfo)
    {
        auto it = setLayoutsInfo.paramsMap.find(name);
        if (it == setLayoutsInfo.paramsMap.end()) 
        {
            MLOGE("Failed write buffer, %s not found!", name.c_str());
            return;
        }

        auto bindInfo = it->second;

        VkWriteDescriptorSet writeDescriptorSet;
        ZeroVulkanStruct(writeDescriptorSet, VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET);
        writeDescriptorSet.dstSet          = descriptorSets[bindInfo.set];
        writeDescriptorSet.descriptorCount = 1;
        writeDescriptorSet.descriptorType  = setLayoutsInfo.GetDescriptorType(bindInfo.set, bindInfo.binding);
        writeDescriptorSet.pBufferInfo     = bufferInfo;
        writeDescriptorSet.dstBinding      = bindInfo.binding;
        vkUpdateDescriptorSets(device, 1, &writeDescriptorSet, 0, nullptr);
    }

    void WriteBuffer(const std::string& name, DVKBuffer* buffer)
    {
        auto it = setLayoutsInfo.paramsMap.find(name);
        if (it == setLayoutsInfo.paramsMap.end()) 
        {
            MLOGE("Failed write buffer, %s not found!", name.c_str());
            return;
        }

        auto bindInfo = it->second;

        VkWriteDescriptorSet writeDescriptorSet;
        ZeroVulkanStruct(writeDescriptorSet, VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET);
        writeDescriptorSet.dstSet          = descriptorSets[bindInfo.set];
        writeDescriptorSet.descriptorCount = 1;
        writeDescriptorSet.descriptorType  = setLayoutsInfo.GetDescriptorType(bindInfo.set, bindInfo.binding);
        writeDescriptorSet.pBufferInfo     = &(buffer->descriptor);
        writeDescriptorSet.dstBinding      = bindInfo.binding;
        vkUpdateDescriptorSets(device, 1, &writeDescriptorSet, 0, nullptr);
    }

public:

    VkDevice	device;

    DVKDescriptorSetLayoutsInfo		setLayoutsInfo;
    std::vector<VkDescriptorSet>	descriptorSets;
};

```

经过了上述的一些列操作之后，我们的整个DescriptorLayout就可以初步的自动化起来。最后我们来改造一下上一个Demo的代码。
**LoadAssets**代码如下：
```c++
void LoadAssets()
{
    // 创建Shader
    m_ShaderTexture = vk_demo::DVKShader::Create(
        m_VulkanDevice, 
        "assets/shaders/16_OptimizeShaderAndLayout/texture.vert.spv",
        "assets/shaders/16_OptimizeShaderAndLayout/texture.frag.spv"
    );
    m_ShaderLut = vk_demo::DVKShader::Create(
        m_VulkanDevice, 
        "assets/shaders/16_OptimizeShaderAndLayout/lut.vert.spv",
        "assets/shaders/16_OptimizeShaderAndLayout/lut.frag.spv"
    );
    m_ShaderLutDebug0 = vk_demo::DVKShader::Create(
        m_VulkanDevice, 
        "assets/shaders/16_OptimizeShaderAndLayout/debug0.vert.spv",
        "assets/shaders/16_OptimizeShaderAndLayout/debug0.frag.spv"
    );
    m_ShaderLutDebug1 = vk_demo::DVKShader::Create(
        m_VulkanDevice, 
        "assets/shaders/16_OptimizeShaderAndLayout/debug1.vert.spv",
        "assets/shaders/16_OptimizeShaderAndLayout/debug1.frag.spv"
    );

    vk_demo::DVKCommandBuffer* cmdBuffer = vk_demo::DVKCommandBuffer::Create(m_VulkanDevice, m_CommandPool);

    // 读取模型文件
    m_Model = vk_demo::DVKModel::LoadFromFile(
        "assets/models/plane_z.obj",
        m_VulkanDevice,
        cmdBuffer,
        m_ShaderTexture->perVertexAttributes
    );
    
    // 生成LUT 3D图数据
    // 64mb 
    // map image0 -> image1
    int32 lutSize  = 256;
    uint8* lutRGBA = new uint8[lutSize * lutSize * 4 * lutSize];
    for (int32 x = 0; x < lutSize; ++x)
    {
        for (int32 y = 0; y < lutSize; ++y)
        {
            for (int32 z = 0; z < lutSize; ++z)
            {
                int idx = (x + y * lutSize + z * lutSize * lutSize) * 4;
                int32 r = x * 1.0f / (lutSize - 1) * 255;
                int32 g = y * 1.0f / (lutSize - 1) * 255;
                int32 b = z * 1.0f / (lutSize - 1) * 255;
                // 怀旧PS滤镜，色调映射。
                r = 0.393f * r + 0.769f * g + 0.189f * b;
                g = 0.349f * r + 0.686f * g + 0.168f * b;
                b = 0.272f * r + 0.534f * g + 0.131f * b;
                lutRGBA[idx + 0] = MMath::Min(r, 255);
                lutRGBA[idx + 1] = MMath::Min(g, 255);
                lutRGBA[idx + 2] = MMath::Min(b, 255);
                lutRGBA[idx + 3] = 255;
            }
        }
    }
    
    // 创建Texture
    m_TexOrigin = vk_demo::DVKTexture::Create2D("assets/textures/game0.jpg", m_VulkanDevice, cmdBuffer);
    m_Tex3DLut  = vk_demo::DVKTexture::Create3D(VK_FORMAT_R8G8B8A8_UNORM, lutRGBA, lutSize * lutSize * 4 * lutSize, lutSize, lutSize, lutSize, m_VulkanDevice, cmdBuffer);
    
    delete cmdBuffer;
}

```
LoadAssets就开始负责Shader的创建。然后DescriptorLayout的创建代码我们就可以整个去掉。最后在调整一下**CreateDescriptorSet**函数，代码如下：
```c++
void CreateDescriptorSet()
{
    m_DescriptorSet0 = m_ShaderTexture->AllocateDescriptorSet();
    m_DescriptorSet0->WriteBuffer("uboMVP", m_MVPBuffer);
    m_DescriptorSet0->WriteImage("diffuseMap", m_TexOrigin);

    m_DescriptorSet1 = m_ShaderLut->AllocateDescriptorSet();
    m_DescriptorSet1->WriteBuffer("uboMVP", m_MVPBuffer);
    m_DescriptorSet1->WriteImage("diffuseMap", m_TexOrigin);
    m_DescriptorSet1->WriteImage("lutMap", m_Tex3DLut);

    m_DescriptorSet2 = m_ShaderLutDebug0->AllocateDescriptorSet();
    m_DescriptorSet2->WriteBuffer("uboMVP", m_MVPBuffer);
    m_DescriptorSet2->WriteImage("diffuseMap", m_TexOrigin);
    m_DescriptorSet2->WriteImage("lutMap", m_Tex3DLut);
    m_DescriptorSet2->WriteBuffer("uboLutDebug", m_LutDebugBuffer);

    m_DescriptorSet3 = m_ShaderLutDebug1->AllocateDescriptorSet();
    m_DescriptorSet3->WriteBuffer("uboMVP", m_MVPBuffer);
    m_DescriptorSet3->WriteImage("diffuseMap", m_TexOrigin);
    m_DescriptorSet3->WriteImage("lutMap", m_Tex3DLut);
    m_DescriptorSet3->WriteBuffer("uboLutDebug", m_LutDebugBuffer);
}
```
目前整个流程将会比之前的更加清晰简洁。代码量也减少了大致100行左右。在后续的Demo里面，我们将不会在进行手动的创建DescriptorLayout数据，将全部的创建工作交给DVKShader，后续也会在Demo里面慢慢补齐缺失的DescriptorLayout数据，后面将不会再着重介绍这个行为。