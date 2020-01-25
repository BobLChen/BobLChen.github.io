---
title: 9_LoadMesh
date: 2019-08-02 19:09:52
tags:
- Vulkan
- 物理设备
- 3D
- Demo
categories:
- Vulkan
---

[LoadMesh](https://github.com/BobLChen/VulkanDemos/tree/master/examples/9_LoadMesh)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

在上一个Demo里面，封装了**VertexBuffer**以及**IndexBuffer**。封装的目的就是为了方便加载外部模型文件显示。这个Demo里面将展示如何加载外部模型并显示。

<!-- more -->

![Preview](https://raw.githubusercontent.com/BobLChen/VulkanTutorials/master/preview/9_LoadMesh.jpg)

## [Assimp](https://github.com/assimp/assimp)

Assimp是一个模型加载库，它支持的模型格式非常多，在以后的Demo里面我们都会使用这个库来加载模型文件。需要注意的是，虽然Assimp能够加载40多种模型格式，但是真正用到的格式不会太多，一般只有几种。例如：**OBJ**、**FBX**、**GLB**、**DAE**、**X**等常规格式。所以。。。我把Assimp的**CMakeLists**配置文件修改了，让它只编译这几种格式的代码。具体详情查看我的提交记录[**点我查看**](https://github.com/BobLChen/VulkanDemos/commit/ecc0254bda2376bd1fa3fb007ad9a619ba4ed310#diff-95dc3f2906f0c1362930db2cc8ddff30)。

## DVKModel封装

在之前的Demo种，使用的是**VertexBuffer**、**IndexBuffer**形式。虽然可以渲染，但是表示起来比较麻烦，加上现在需要集成**Assimp**，使用**Assimp**来加载。那么就有必要封装出一个**DVKModel**用来加载外部模型、同时存储模型的相关信息。

### DVKBoundingBox

我们首先封装一个Bounds出来，用来表示模型的尺寸，这个有助于加载完模型之后适配相机。

```c++
struct DVKBoundingBox
    {
        Vector3 min;
        Vector3 max;
        
        DVKBoundingBox()
            : min(MAX_flt, MAX_flt, MAX_flt)
            , max(MIN_flt, MIN_flt, MIN_flt)
        {
            
        }
        
        DVKBoundingBox(const Vector3& inMin, const Vector3& inMax)
            : min(inMin)
            , max(inMax)
        {
            
        }
    };
```

### DVKPrimitive

在上一节里面，我们封装过**IndexBuffer**，其实我有提到过一些低端设备的索引只支持**UInt16**，也就是**65535**个索引数据，也就是**21,845‬**个三角形。但是一个模型的面数不可能只低于21845，是极有可能超过21845的。当遇到模型面数超过21845的时候，需要拆分它们。为了这个功能，设计出**DVKPrimitive**，用来表示一个网格，多个网格组成一个模型。

```c++
struct DVKPrimitive
	{
		DVKIndexBuffer*					indexBuffer = nullptr;
        DVKVertexBuffer*				vertexBuffer = nullptr;

		std::vector<float>				vertices;
		std::vector<uint16>				indices;
        
        int32                           vertexCount = 0;
        int32                           indexCount = 0;

		DVKPrimitive()
		{

		}
        
		~DVKPrimitive()
		{
			if (indexBuffer) {
				delete indexBuffer;
			}
			if (vertexBuffer) {
				delete vertexBuffer;
			}
			indexBuffer  = nullptr;
			vertexBuffer = nullptr;
		}

		void BindDrawCmd(VkCommandBuffer cmdBuffer)
		{
			vertexBuffer->Bind(cmdBuffer);
			indexBuffer->BindDraw(cmdBuffer);
		}
	};
```

在这个网格里面，存储了真正的IndexBuffer以及VertexBuffer。

### DVKMesh

有了**DVKPrimitive**之后，就可以非常方便的设计出Mesh的结构。Mesh的结构没有很特殊，它包含了一组**DVKPrimitive**数据。另外有一个地方需要注意的是，外部模型文件可能是代表的一个完整的场景，这个场景里面是通过众多的节点及其子集组合而成。这些节点可能代表的含义不同，例如有**Camera**、**Light**、**Mesh**、**Bone**等等。

其实表示它们最清晰的结构应该是：

```
class Object3D
{
	
};

class Camera : public Object3D
{
	
};

class Light : public Object3D
{

};
```

但是呢，在这个Demo里面，我不准备把这些信息也导入进来，只关心模型数据，其它的在Demo内部创建。所以我把Mesh的设计成了如下结构。让Mesh去关联一个**Node**，**Node**存储一些基本信息，例如位移、缩放、旋转、名称、子父集关系等。

```c++
struct DVKMesh
    {
		typedef std::vector<DVKPrimitive*> DVKPrimitives;

		DVKPrimitives	primitives;
		DVKBoundingBox	bounding;
        DVKNode*		linkNode;

		DVKMaterialInfo	material;
        
		int32			vertexCount;
		int32			triangleCount;

        DVKMesh()
            : linkNode(nullptr)
			, vertexCount(0)
			, triangleCount(0)
        {
            
        }

		void BindDrawCmd(VkCommandBuffer cmdBuffer)
		{
			for (int i = 0; i < primitives.size(); ++i) {
				primitives[i]->BindDrawCmd(cmdBuffer);
			}
		}
        
        ~DVKMesh()
        {
			for (int i = 0; i < primitives.size(); ++i) {
				delete primitives[i];
			}
            primitives.clear();
            linkNode = nullptr;
        }
    };
```

### DVKNode

综上所述，还需要一个Node用来保存节点的一些基本信息。Node的设计如下：

```c++
struct DVKNode
    {
        std::string					name;

		std::vector<DVKMesh*>		meshes;

		DVKNode*					parent;
        std::vector<DVKNode*>		children;

        Matrix4x4					localMatrix;
        Matrix4x4					globalMatrix;

		int32						index;
        
        DVKNode()
            : name("None")
            , parent(nullptr)
			, index(-1)
        {
            
        }
        
        const Matrix4x4& GetLocalMatrix()
        {
            return localMatrix;
        }
        
        Matrix4x4& GetGlobalMatrix()
        {
            globalMatrix = localMatrix;
            
            if (parent) {
                globalMatrix.Append(parent->GetGlobalMatrix());
            }
            return globalMatrix;
        }

		DVKBoundingBox GetBounds()
		{
			DVKBoundingBox bounds;
			bounds.min.Set(0, 0, 0);
			bounds.max.Set(0, 0, 0);

			if (meshes.size() > 0) 
			{
				for (int32 i = 0; i < meshes.size(); ++i)
				{
					const Matrix4x4& matrix = GetGlobalMatrix();
					Vector3 mmin = matrix.TransformPosition(meshes[i]->bounding.min);
					Vector3 mmax = matrix.TransformPosition(meshes[i]->bounding.max);

					bounds.min.x = MMath::Min(bounds.min.x, mmin.x);
					bounds.min.y = MMath::Min(bounds.min.y, mmin.y);
					bounds.min.z = MMath::Min(bounds.min.z, mmin.z);

					bounds.max.x = MMath::Max(bounds.max.x, mmax.x);
					bounds.max.y = MMath::Max(bounds.max.y, mmax.y);
					bounds.max.z = MMath::Max(bounds.max.z, mmax.z);
				}
			}

			for (int32 i = 0; i < children.size(); ++i) 
			{
				DVKBoundingBox childBounds = children[i]->GetBounds();
				bounds.min.x = MMath::Min(bounds.min.x, childBounds.min.x);
				bounds.min.y = MMath::Min(bounds.min.y, childBounds.min.y);
				bounds.min.z = MMath::Min(bounds.min.z, childBounds.min.z);

				bounds.max.x = MMath::Max(bounds.max.x, childBounds.max.x);
				bounds.max.y = MMath::Max(bounds.max.y, childBounds.max.y);
				bounds.max.z = MMath::Max(bounds.max.z, childBounds.max.z);
			}

			return bounds;
		}
        
        ~DVKNode()
        {
			for (int32 i = 0; i < meshes.size(); ++i) {
				delete meshes[i];
			}
			meshes.clear();
            
            for (int32 i = 0; i < children.size(); ++i) {
                delete children[i];
            }
            children.clear();
        }
    };
```

其中Node提供两个关键的API，一个是可以获取**Global**的**矩阵**信息，一个是可以获取该节点的Bounds信息。

## DVKModel

有了基本的信息之后，在Model中把它们组合起来。

```c++
class DVKModel
    {
    private:
        DVKModel()
			: device(nullptr)
			, rootNode(nullptr)
        {
            
        }
        
    public:
        ~DVKModel()
        {
            delete rootNode;
			rootNode = nullptr;
			device = nullptr;

			meshes.clear();
			linearNodes.clear();
        }

		VkVertexInputBindingDescription GetInputBinding();

		std::vector<VkVertexInputAttributeDescription> GetInputAttributes();
        
        static DVKModel* LoadFromFile(const std::string& filename, std::shared_ptr<VulkanDevice> vulkanDevice, DVKCommandBuffer* cmdBuffer, const std::vector<VertexAttribute>& attributes);
        
        static DVKModel* Create(std::shared_ptr<VulkanDevice> vulkanDevice, DVKCommandBuffer* cmdBuffer, const std::vector<float>& vertices, const std::vector<uint16>& indices, const std::vector<VertexAttribute>& attributes);
        
    protected:
        
        DVKNode* LoadNode(const aiNode* node, const aiScene* scene);
        
		DVKMesh* LoadMesh(const aiMesh* mesh, const aiScene* scene);

    public:
        
        std::shared_ptr<VulkanDevice>	device;
        
        DVKNode*						rootNode;
        std::vector<DVKNode*>			linearNodes;
        std::vector<DVKMesh*>			meshes;

		std::vector<VertexAttribute>	attributes;

	private:

		DVKCommandBuffer*				cmdBuffer;
    };
```

**DVKModel**需要有如下几个功能：

- 能够解析**Assimp**加载的数据
- 能加载本地文件
- 能够从内存中加载
- 能够制定数据结构

### LoadFromFile

首要任务是能够从本地加载模型文件并解析出来。通过Assimp库即可完成加载解析功能，但是结构的解析还是得需要我们自己完成。**LoadFromFile**函数如下：

```c++
DVKModel* DVKModel::LoadFromFile(const std::string& filename, std::shared_ptr<VulkanDevice> vulkanDevice, DVKCommandBuffer* cmdBuffer, const std::vector<VertexAttribute>& attributes)
    {
        DVKModel* model   = new DVKModel();
        model->device     = vulkanDevice;
		model->attributes = attributes;
		model->cmdBuffer  = cmdBuffer;
        
        int assimpFlags =
            aiProcess_Triangulate |
            aiProcess_MakeLeftHanded |
            aiProcess_FlipUVs |
            aiProcess_FlipWindingOrder;
        
        for (int32 i = 0; i < attributes.size(); ++i) {
            if (attributes[i] == VertexAttribute::VA_Tangent) {
                assimpFlags = assimpFlags | aiProcess_CalcTangentSpace;
            }
            else if (attributes[i] == VertexAttribute::VA_UV0) {
                assimpFlags = assimpFlags | aiProcess_GenUVCoords;
            }
            else if (attributes[i] == VertexAttribute::VA_Normal) {
                assimpFlags = assimpFlags | aiProcess_GenSmoothNormals;
            }
        }

        uint32 dataSize = 0;
        uint8* dataPtr  = nullptr;
        if (!FileManager::ReadFile(filename, dataPtr, dataSize)) {
            return model;
        }
        
        Assimp::Importer importer;
        const aiScene* scene = importer.ReadFileFromMemory(dataPtr, dataSize, assimpFlags);
        
		model->LoadNode(scene->mRootNode, scene);

        return model;
    }
```

这里需要注意的是**assimpFlags**的配置

- **aiProcess_Triangulate**：表示需要将模型三角化，因为模型可能是由四边形组成的。
- **aiProcess_MakeLeftHanded**：表示需要左手坐标系。
- **aiProcess_FlipUVs**：表示希望翻转UV坐标中的Y轴。
- **aiProcess_FlipWindingOrder**：表示需要翻转索引顺序。

千万千万不要加**aiProcess_PreTransformVertices**这个属性，这个属性的功能是把你的顶点数据转化到世界空间。转化到了世界空间之后，相当于所有的Mesh都是在坐标原点，且没有缩放，没有旋转。这个不利于我们学习，我们需要的是Mesh被摆放到世界空间中。

另外就是，我们需要遍历一下传入的顶点数据类型，如果有**Normal**、**Tangent**、**UV**，需要把**aiProcess_GenXXX**加上，因为模型里面可能不包含这些数据，**Assimp**库是可以自动生成这些数据的。

我们最好使用**Assimp**的**ReadFileFromMemory**接口，尽量不要使用**LoadFromFile**接口。考虑到后面可能会在**Android**等平台上面使用，但是这些平台的文件读取是有特有的接口，为了支持这些平台需要实现平台相关的文件操作功能。所以使用它的**ReadFileFromMemory**接口即可。

至于后面的就很简单了，读取文件，通过Assimp解析。。。

### LoadNode

有了**Assimp**加载好的数据之后，从Root节点开始递归遍历，逐个解析即可。

```c++
DVKNode* DVKModel::LoadNode(const aiNode* aiNode, const aiScene* aiScene)
    {
		DVKNode* vkNode = new DVKNode();
		vkNode->name = aiNode->mName.C_Str();

		if (rootNode == nullptr) {
			rootNode = vkNode;
		}

		// local matrix
		vkNode->localMatrix.m[0][0] = aiNode->mTransformation.a1;
		vkNode->localMatrix.m[0][1] = aiNode->mTransformation.a2;
		vkNode->localMatrix.m[0][2] = aiNode->mTransformation.a3;
		vkNode->localMatrix.m[0][3] = aiNode->mTransformation.a4;
		vkNode->localMatrix.m[1][0] = aiNode->mTransformation.b1;
		vkNode->localMatrix.m[1][1] = aiNode->mTransformation.b2;
		vkNode->localMatrix.m[1][2] = aiNode->mTransformation.b3;
		vkNode->localMatrix.m[1][3] = aiNode->mTransformation.b4;
		vkNode->localMatrix.m[2][0] = aiNode->mTransformation.c1;
		vkNode->localMatrix.m[2][1] = aiNode->mTransformation.c2;
		vkNode->localMatrix.m[2][2] = aiNode->mTransformation.c3;
		vkNode->localMatrix.m[2][3] = aiNode->mTransformation.c4;
		vkNode->localMatrix.m[3][0] = aiNode->mTransformation.d1;
		vkNode->localMatrix.m[3][1] = aiNode->mTransformation.d2;
		vkNode->localMatrix.m[3][2] = aiNode->mTransformation.d3;
		vkNode->localMatrix.m[3][3] = aiNode->mTransformation.d4;
        vkNode->localMatrix.SetTransposed();
        
		// mesh
        if (aiNode->mNumMeshes > 0) {
			for (int i = 0; i < aiNode->mNumMeshes; ++i) 
			{
				DVKMesh* vkMesh = LoadMesh(aiScene->mMeshes[aiNode->mMeshes[i]], aiScene);
				vkMesh->linkNode = vkNode;
				meshes.push_back(vkMesh);
				vkNode->meshes.push_back(vkMesh);
			}
        }
        
		linearNodes.push_back(vkNode);
		// children node
        for (int32 i = 0; i < aiNode->mNumChildren; ++i) 
		{
            DVKNode* childNode = LoadNode(aiNode->mChildren[i], aiScene);
			childNode->parent  = vkNode;
			vkNode->children.push_back(childNode);
        }
        
		return vkNode;
    }
```

### LoadMesh

LoadMesh是在访问**Node**节点时触发的，如果Node节点含有Mesh，就通过LoadMesh函数从中读取出Mesh数据。LoadMesh需要注意几个地方：1、数据的存储格式要跟定义的格式保持一致；2、遇到面数过多的Mesh需要进行拆分。

```c++
DVKMesh* DVKModel::LoadMesh(const aiMesh* aiMesh, const aiScene* aiScene)
	{
		DVKMesh* mesh = new DVKMesh();

		Vector3 mmin(MAX_flt, MAX_flt, MAX_flt);
		Vector3 mmax(MIN_flt, MIN_flt, MIN_flt);

		std::vector<float>  vertices;
		std::vector<uint32> indices;

		aiMaterial* material = aiScene->mMaterials[aiMesh->mMaterialIndex];
		if (material) {
			FillMaterialTextures(material, mesh->material);
		}

		aiString texPath;
		material->GetTexture(aiTextureType::aiTextureType_DIFFUSE, 0, &texPath);

		Vector3 defaultColor = Vector3(MMath::RandRange(0.0f, 1.0f), MMath::RandRange(0.0f, 1.0f), MMath::RandRange(0.0f, 1.0f));

		for (int32 i = 0; i < aiMesh->mNumVertices; ++i)
		{
			for (int32 j = 0; j < attributes.size(); ++j)
			{
				if (attributes[j] == VertexAttribute::VA_Position) 
				{
					float v0 = aiMesh->mVertices[i].x;
					float v1 = aiMesh->mVertices[i].y;
					float v2 = aiMesh->mVertices[i].z;

					vertices.push_back(v0);
					vertices.push_back(v1);
					vertices.push_back(v2);

					mmin.x = MMath::Min(v0, mmin.x);
					mmin.y = MMath::Min(v1, mmin.y);
					mmin.z = MMath::Min(v2, mmin.z);
					mmax.x = MMath::Max(v0, mmax.x);
					mmax.y = MMath::Max(v1, mmax.y);
					mmax.z = MMath::Max(v2, mmax.z);
				}
				else if (attributes[j] == VertexAttribute::VA_UV0) 
				{
					vertices.push_back(aiMesh->mTextureCoords[0][i].x);
					vertices.push_back(aiMesh->mTextureCoords[0][i].y);
				}
				else if (attributes[j] == VertexAttribute::VA_UV1) 
				{
					vertices.push_back(aiMesh->mTextureCoords[1][i].x);
					vertices.push_back(aiMesh->mTextureCoords[1][i].y);
				}
				else if (attributes[j] == VertexAttribute::VA_Normal) 
				{
					vertices.push_back(aiMesh->mNormals[i].x);
					vertices.push_back(aiMesh->mNormals[i].y);
					vertices.push_back(aiMesh->mNormals[i].z);
				}
				else if (attributes[j] == VertexAttribute::VA_Tangent) 
				{
					vertices.push_back(aiMesh->mTangents[i].x);
					vertices.push_back(aiMesh->mTangents[i].y);
					vertices.push_back(aiMesh->mTangents[i].z);
					vertices.push_back(1);
				}
				else if (attributes[j] == VertexAttribute::VA_Color) 
				{
					if (aiMesh->HasVertexColors(i))
					{
						vertices.push_back(aiMesh->mColors[0][i].r);
						vertices.push_back(aiMesh->mColors[0][i].g);
						vertices.push_back(aiMesh->mColors[0][i].b);
					}
					else 
					{
						vertices.push_back(defaultColor.x);
						vertices.push_back(defaultColor.y);
						vertices.push_back(defaultColor.z);
					}
				}
                else if (attributes[j] == VertexAttribute::VA_Custom0 ||
                         attributes[j] == VertexAttribute::VA_Custom1 ||
                         attributes[j] == VertexAttribute::VA_Custom2 ||
                         attributes[j] == VertexAttribute::VA_Custom3
                )
                {
                    vertices.push_back(0.0f);
                    vertices.push_back(0.0f);
                    vertices.push_back(0.0f);
                    vertices.push_back(0.0f);
                }
			}
		}
        
		for (int32 i = 0; i < aiMesh->mNumFaces; ++i)
		{
			indices.push_back(aiMesh->mFaces[i].mIndices[0]);
			indices.push_back(aiMesh->mFaces[i].mIndices[1]);
			indices.push_back(aiMesh->mFaces[i].mIndices[2]);
		}

		int32 stride = vertices.size() / aiMesh->mNumVertices;

		if (indices.size() > 65535) 
		{
			std::unordered_map<uint32, uint32> indicesMap;
			DVKPrimitive* primitive = nullptr;

			for (int32 i = 0; i < indices.size(); ++i) {
				uint32 idx = indices[i];
				if (primitive == nullptr) {
					primitive = new DVKPrimitive();
					indicesMap.clear();
					mesh->primitives.push_back(primitive);
				}

				uint32 newIdx = 0;
				auto it = indicesMap.find(idx);
				if (it == indicesMap.end()) 
				{
					uint32 start = idx * stride;
					newIdx = primitive->vertices.size() / stride;
					primitive->vertices.insert(primitive->vertices.end(), vertices.begin() + start, vertices.begin() + start + stride);
					indicesMap.insert(std::make_pair(idx, newIdx));
				}
				else
				{
					newIdx = it->second;
				}

				primitive->indices.push_back(newIdx);

				if (primitive->indices.size() == 65535) {
					primitive = nullptr;
				}
			}

            if (cmdBuffer)
            {
                for (int32 i = 0; i < mesh->primitives.size(); ++i)
                {
                    primitive = mesh->primitives[i];
                    primitive->vertexBuffer = DVKVertexBuffer::Create(device, cmdBuffer, primitive->vertices, attributes);
                    primitive->indexBuffer  = DVKIndexBuffer::Create(device, cmdBuffer, primitive->indices);
                }
            }
		}
		else
		{
			DVKPrimitive* primitive = new DVKPrimitive();
			primitive->vertices = vertices;
			for (uint16 i = 0; i < indices.size(); ++i) {
				primitive->indices.push_back(indices[i]);
			}
			mesh->primitives.push_back(primitive);
            
            if (cmdBuffer)
            {
                primitive->vertexBuffer = DVKVertexBuffer::Create(device, cmdBuffer, primitive->vertices, attributes);
                primitive->indexBuffer  = DVKIndexBuffer::Create(device, cmdBuffer, primitive->indices);
            }
		}
        
		for (int32 i = 0; i < mesh->primitives.size(); ++i)
		{
            DVKPrimitive* primitive = mesh->primitives[i];
            primitive->vertexCount  = primitive->vertices.size() / stride;
            primitive->indexCount   = primitive->indices.size() / 3;
            
            mesh->vertexCount   += primitive->vertexCount;
            mesh->triangleCount += primitive->indexCount;
		}
        
		mesh->bounding.min = mmin;
		mesh->bounding.max = mmax;

		return mesh;
	}
```

自此就完成了模型的解析工作。

## Usage

有了封装之后的DVKModel之后，就可以着手修改Demo，把之前的三角形替换为外部的模型。如下所示：

```c++
void LoadAssets()
	{
		vk_demo::DVKCommandBuffer* cmdBuffer = vk_demo::DVKCommandBuffer::Create(m_VulkanDevice, m_CommandPool);

		m_Model = vk_demo::DVKModel::LoadFromFile(
			"assets/models/Vela_Template.fbx",
			m_VulkanDevice,
			cmdBuffer,
			{ VertexAttribute::VA_Position, VertexAttribute::VA_Normal }
		);

		delete cmdBuffer;
	}
```

考虑Model中可能含有多个Mesh，那么这些Mesh对应的**MVP**矩阵就需要有多份。每一份**MVP**矩阵代表一个**Mesh**。那么**CreateUniformBuffers**修改如下：

```c++
void CreateUniformBuffers()
	{
		vk_demo::DVKBoundingBox bounds = m_Model->rootNode->GetBounds();
		Vector3 boundSize   = bounds.max - bounds.min;
        Vector3 boundCenter = bounds.min + boundSize * 0.5f;
		boundCenter.z -= boundSize.Size();

		m_MVPDatas.resize(m_Model->meshes.size());
		m_MVPBuffers.resize(m_Model->meshes.size());

		for (int32 i = 0; i < m_Model->meshes.size(); ++i)
		{
			m_MVPDatas[i].model.SetIdentity();
			m_MVPDatas[i].model.SetOrigin(Vector3(0, 0, 0));
        
			m_MVPDatas[i].view.SetIdentity();
			m_MVPDatas[i].view.SetOrigin(boundCenter);
			m_MVPDatas[i].view.SetInverse();

			m_MVPDatas[i].projection.SetIdentity();
			m_MVPDatas[i].projection.Perspective(MMath::DegreesToRadians(75.0f), (float)GetWidth(), (float)GetHeight(), 0.01f, 3000.0f);
		
			m_MVPBuffers[i] = vk_demo::DVKBuffer::CreateBuffer(
				m_VulkanDevice, 
				VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, 
				VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, 
				sizeof(UBOData),
				&(m_MVPDatas[i])
			);
			m_MVPBuffers[i]->Map();
		}
	}
```

既然**UniformBuffer**有了多个，那么回想之前的Demo，**UniformBuffer**是通过**DescriptorSet**进行关联的，显然一个**DescriptorSet**无法关联这么多**Buffer**(DescriptorSet的更新不能发生在CommandBuffer录制期间)。那么**DescriptorSet**也需要创建多个。

先扩充Pool的尺寸。

```c++
void CreateDescriptorPool()
	{
		VkDescriptorPoolSize poolSize = {};
		poolSize.type            = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
		poolSize.descriptorCount = 1;
        
		VkDescriptorPoolCreateInfo descriptorPoolInfo;
		ZeroVulkanStruct(descriptorPoolInfo, VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO);
		descriptorPoolInfo.poolSizeCount = 1;
		descriptorPoolInfo.pPoolSizes    = &poolSize;
		descriptorPoolInfo.maxSets       = m_Model->meshes.size();
		VERIFYVULKANRESULT(vkCreateDescriptorPool(m_Device, &descriptorPoolInfo, VULKAN_CPU_ALLOCATOR, &m_DescriptorPool));
	}
```

然后创建Set

```c++
void CreateDescriptorSet()
	{
		m_DescriptorSets.resize(m_Model->meshes.size());
		for (int32 i = 0; i < m_DescriptorSets.size(); ++i)
		{
			VkDescriptorSet descriptorSet = VK_NULL_HANDLE;

			VkDescriptorSetAllocateInfo allocInfo;
			ZeroVulkanStruct(allocInfo, VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO);
			allocInfo.descriptorPool     = m_DescriptorPool;
			allocInfo.descriptorSetCount = 1;
			allocInfo.pSetLayouts        = &m_DescriptorSetLayout;
			VERIFYVULKANRESULT(vkAllocateDescriptorSets(m_Device, &allocInfo, &descriptorSet));
        
			VkWriteDescriptorSet writeDescriptorSet;
			ZeroVulkanStruct(writeDescriptorSet, VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET);
			writeDescriptorSet.dstSet          = descriptorSet;
			writeDescriptorSet.descriptorCount = 1;
			writeDescriptorSet.descriptorType  = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
			writeDescriptorSet.pBufferInfo     = &(m_MVPBuffers[i]->descriptor);
			writeDescriptorSet.dstBinding      = 0;
			vkUpdateDescriptorSets(m_Device, 1, &writeDescriptorSet, 0, nullptr);

			m_DescriptorSets[i] = descriptorSet;
		}
	}
```

最好稍微修改一下录制命令的代码：

```c++
void SetupCommandBuffers()
	{
		VkCommandBufferBeginInfo cmdBeginInfo;
		ZeroVulkanStruct(cmdBeginInfo, VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO);

		VkClearValue clearValues[2];
		clearValues[0].color        = { {0.2f, 0.2f, 0.2f, 1.0f} };
		clearValues[1].depthStencil = { 1.0f, 0 };

		VkRenderPassBeginInfo renderPassBeginInfo;
		ZeroVulkanStruct(renderPassBeginInfo, VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO);
        renderPassBeginInfo.renderPass      = m_RenderPass;
		renderPassBeginInfo.clearValueCount = 2;
		renderPassBeginInfo.pClearValues    = clearValues;
		renderPassBeginInfo.renderArea.offset.x = 0;
		renderPassBeginInfo.renderArea.offset.y = 0;
        renderPassBeginInfo.renderArea.extent.width  = m_FrameWidth;
        renderPassBeginInfo.renderArea.extent.height = m_FrameHeight;
        
		for (int32 i = 0; i < m_CommandBuffers.size(); ++i)
		{
            renderPassBeginInfo.framebuffer = m_FrameBuffers[i];
            
			VkViewport viewport = {};
			viewport.x        = 0;
			viewport.y        = m_FrameHeight;
            viewport.width    = (float)m_FrameWidth;
            viewport.height   = -(float)m_FrameHeight;    // flip y axis
			viewport.minDepth = 0.0f;
			viewport.maxDepth = 1.0f;
            
			VkRect2D scissor = {};
            scissor.extent.width  = m_FrameWidth;
            scissor.extent.height = m_FrameHeight;
			scissor.offset.x      = 0;
			scissor.offset.y      = 0;
            
			VERIFYVULKANRESULT(vkBeginCommandBuffer(m_CommandBuffers[i], &cmdBeginInfo));
            
			vkCmdBeginRenderPass(m_CommandBuffers[i], &renderPassBeginInfo, VK_SUBPASS_CONTENTS_INLINE);
			vkCmdSetViewport(m_CommandBuffers[i], 0, 1, &viewport);
			vkCmdSetScissor(m_CommandBuffers[i], 0, 1, &scissor);

			vkCmdBindPipeline(m_CommandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, m_Pipeline);

			for (int32 meshIndex = 0; meshIndex < m_Model->meshes.size(); ++meshIndex)
			{
				vkCmdBindDescriptorSets(m_CommandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, m_PipelineLayout, 0, 1, &m_DescriptorSets[meshIndex], 0, nullptr);
				m_Model->meshes[meshIndex]->BindDrawCmd(m_CommandBuffers[i]);
			}
			
			m_GUI->BindDrawCmd(m_CommandBuffers[i], m_RenderPass);

			vkCmdEndRenderPass(m_CommandBuffers[i]);
            
			VERIFYVULKANRESULT(vkEndCommandBuffer(m_CommandBuffers[i]));
		}
	}
```

自此我们就完成了加载外部模型文件并显示的需求。无论是当模型亦或者说一个非常复杂的场景，在这个Demo里面都可以正确的加载并显示出来。